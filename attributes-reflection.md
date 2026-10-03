# Attributes & Reflection

> **Baseline:** .NET **10** / C# **14**. Generic attributes có từ C# 11; các annotation trimming/AOT trong chương này có sẵn trên baseline.

Attribute gắn metadata vào declaration; reflection đọc thông tin kiểu và member, tạo object hoặc gọi method lúc chạy. Compiler, analyzer, source generator và framework có thể dùng cùng metadata theo những cách khác nhau. [Tổng quan Microsoft](https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/reflection-and-attributes/).

## Mục lục

- [1. Metadata và hành vi](#1-metadata-và-hành-vi)
- [2. Custom attribute và AttributeUsage](#2-custom-attribute-và-attributeusage)
  - [2.1 Attribute có consumer sẵn](#21-attribute-có-consumer-sẵn)
- [3. Target và đối số attribute](#3-target-và-đối-số-attribute)
- [4. Đọc attribute và kế thừa](#4-đọc-attribute-và-kế-thừa)
- [5. Type, member và BindingFlags](#5-type-member-và-bindingflags)
- [6. Invoke và CreateDelegate](#6-invoke-và-createdelegate)
- [7. Generic reflection](#7-generic-reflection)
- [8. Nullable metadata](#8-nullable-metadata)
- [9. Trimming và Native AOT](#9-trimming-và-native-aot)
- [10. Chọn cách triển khai](#10-chọn-cách-triển-khai)
- [11. Interface, event, field và đối số](#11-interface-event-field-và-đối-số)

---

## 1. Metadata và hành vi

Assembly chứa metadata về type, member, signature và custom attributes. Attribute tự định nghĩa cần một consumer đọc nó; gắn `[Audit]` lên method chưa tự tạo logging. Một số attribute đã có consumer: compiler xử lý `[Obsolete]`, serializer đọc `[JsonPropertyName]`, analyzer đọc `[DynamicallyAccessedMembers]`.

```csharp
using System;
using System.Reflection;

Type type = typeof(string);
Assembly assembly = type.Assembly;
Console.WriteLine(type.FullName);
Console.WriteLine(assembly.GetName().Name);
```

`typeof(T)` lấy type từ declaration; `obj.GetType()` lấy concrete runtime type và cần `obj` khác null. `nameof(T)` chỉ tạo tên ở compile-time. Xem [hệ thống kiểu](typesystem.md#13-clr-il-metadata) và [toán tử](operators.md).

`Type.GetType(string)` tra tên lúc chạy. Tên không assembly-qualified chỉ tìm assembly đang gọi và `System.Runtime` / corelib. Không thấy thì trả `null`; `throwOnError: true` mới ném. Chuỗi do người dùng đưa vào không phải danh sách type an toàn — trimmer không giữ type chỉ vì tên xuất hiện trong literal.

```csharp
Type? list = Type.GetType("System.Collections.Generic.List`1"); // arity `1`, có thể null
Type? exact = Type.GetType(
    "System.Text.Json.JsonSerializer, System.Text.Json",
    throwOnError: false);
```

`Assembly.GetTypes()` ném `ReflectionTypeLoadException` khi một type trong assembly không load được. Các type load được nằm trong `Types`; nguyên nhân từng type nằm trong `LoaderExceptions`. `GetExportedTypes()` chỉ public type, cùng kiểu lỗi. Duyệt assembly plugin trên Native AOT không thay `Assembly.Load` của IL tùy ý — mục 9.

## 2. Custom attribute và AttributeUsage

```csharp
using System;

[AttributeUsage(
    AttributeTargets.Class | AttributeTargets.Method,
    AllowMultiple = true,
    Inherited = true)]
public sealed class AuditAttribute : Attribute
{
    public AuditAttribute(string category) => Category = category;

    public string Category { get; }
    public bool Enabled { get; set; } = true;
}

[Audit("billing", Enabled = false)]
public class InvoiceService
{
    [Audit("write")]
    [Audit("business")]
    public virtual void Save() { }
}
```

`Audit` và `AuditAttribute` cùng resolve về attribute ở trên. `"billing"` là đối số constructor; `Enabled = false` là named argument gán public writable property sau khi tạo instance.

| Thiết lập | Ý nghĩa | Mặc định khi không khai báo AttributeUsage |
|---|---|---|
| `AttributeTargets` | Các declaration được phép gắn attribute; kết hợp bằng `\|` | `All` |
| `AllowMultiple` | Cho phép lặp cùng attribute trên một declaration | `false` |
| `Inherited` | Cho phép truy vấn kế thừa attribute từ base class / overridden member | `true` |

`Inherited` không làm class attribute tự trở thành base class của attribute khác. Khi `AllowMultiple = false`, attribute khai báo ở derived có thể thay thế attribute cùng loại từ base trong kết quả truy vấn kế thừa. [Custom attributes](https://learn.microsoft.com/en-us/dotnet/standard/attributes/writing-custom-attributes).

### 2.1 Attribute có consumer sẵn

Gắn attribute chỉ có tác dụng khi một consumer đọc nó. Compiler, analyzer và runtime đã đọc một số attribute sau; attribute tự viết không nằm trong bảng này.

| Attribute | Consumer | Việc xảy ra |
|---|---|---|
| `[Obsolete("…", error: false)]` | Compiler | Cảnh báo ở chỗ dùng. `error: true` thành lỗi biên dịch. Không chặn reflection gọi method |
| `[Conditional("DEBUG")]` | Compiler | Gỡ lời gọi trực tiếp khi symbol tắt. Method vẫn còn trong assembly. Chi tiết [preprocessor §5](preprocessor-directives.md#5-conditionalattribute-vs-preprocessor) |
| `[CallerMemberName]` / `[CallerFilePath]` / `[CallerLineNumber]` / `[CallerArgumentExpression]` | Compiler | Điền optional parameter ở call site. Không phải thông tin runtime của caller qua stack |
| `[ModuleInitializer]` | Runtime | Gọi method `static void` không tham số, không generic, trước entry. [thứ tự Main](main-function.md#13-thứ-tự-khởi-động-console) |
| `[Experimental("DIAG001")]` | Compiler (.NET 8+) | Chỗ dùng phải xử lý diagnostic id đó (`#pragma` / `SuppressMessage`), kể cả trong cùng assembly |
| `[OverloadResolutionPriority(n)]` | Compiler (C# 13) | Số lớn hơn thắng khi nhiều overload cùng applicable. Không đổi chữ ký lúc chạy. [methods](methods.md) |
| `[SetsRequiredMembers]` | Compiler | Constructor được coi là đã gán mọi `required`. Gắn sai thì caller bỏ qua member bắt buộc. [required](oop.md#54-init-only-c-9--required-c-11) |

```csharp
public static class Guard
{
    public static void ThrowIfNull(
        object? value,
        [CallerArgumentExpression(nameof(value))] string? expression = null)
    {
        if (value is null)
            throw new ArgumentNullException(expression);
    }
}

// ThrowIfNull(user); → expression nhận chuỗi "user", không phải tên parameter "value".
```

Caller attribute chỉ điền khi argument đó **vắng** ở call site. Truyền `expression: "other"` thì compiler không ghi đè. Parameter phải có default. `[CallerArgumentExpression(nameof(value))]` trỏ tới parameter **khác** trong cùng method — chuỗi là biểu thức nguồn, không phải giá trị.

`[ModuleInitializer]` chạy lại mỗi lần load module, kể cả test host. Đừng mở socket hay đọc config có thể fail: lỗi ở đây làm assembly không dùng được, trước `Main`. Method phải `internal` hoặc `public`, `void`, không `async`.

`[Obsolete]` không ẩn member khỏi `GetMethods`. Code reflection và source generator vẫn thấy method cũ. Muốn xóa khỏi contract công khai thì bỏ method, không chỉ đánh obsolete.

## 3. Target và đối số attribute

### 3.1 Target tường minh

```csharp
using System.Diagnostics.CodeAnalysis;
using System.Text.Json.Serialization;

// Gắn vào property được compiler sinh cho positional record.
public sealed record Customer(
    [property: JsonPropertyName("customer_id")] string Id);

public static class TextHelpers
{
    [return: NotNullIfNotNull(nameof(value))]
    public static string? Trim(string? value) => value?.Trim();
}
```

| Target | Nơi gắn metadata |
|---|---|
| `assembly:` / `module:` | Assembly / module; đặt declaration này sau using và trước type declarations |
| `return:` | Giá trị trả về; reflection đọc qua `MethodInfo.ReturnParameter` |
| `param:` | Parameter, gồm primary constructor parameter |
| `property:` | Property; hữu ích với positional record |
| `field:` | Field, gồm backing field của auto-property |
| `method:` | Method; có thể dùng trên declaration có method do compiler sinh nếu target đó hợp lệ |
| `type:` | Type declaration |

Attribute vẫn phải cho phép target đó qua `AttributeUsage`. Target `property:` trên primary constructor parameter của class thường không tạo property; positional record có cơ chế sinh property riêng. Xem [primary constructor](oop.md#19-primary-constructor-c-12).

Đọc lại đúng chỗ metadata đã ghi. `return:` không nằm trên `MethodInfo` như attribute của method.

```csharp
MethodInfo trim = typeof(TextHelpers).GetMethod(nameof(TextHelpers.Trim))!;
ParameterInfo ret = trim.ReturnParameter;
bool onReturn = ret.GetCustomAttributesData()
    .Any(data => data.AttributeType.Name.Contains("NotNullIfNotNull", StringComparison.Ordinal));

ParameterInfo value = trim.GetParameters().Single();
// value.GetCustomAttributes(...) đọc attribute của parameter, không phải của method.
_ = onReturn;
```

`[assembly: InternalsVisibleTo("MyApp.Tests")]` là attribute cấp assembly, đặt sau `using`, ngoài namespace. Khớp tên assembly và strong name: [projects-packages §8](projects-packages.md#8-internalsvisibleto). `GetCustomAttributes` trên `Assembly` không dùng cờ `inherit`.

`[field: ...]` trên auto-property ghi vào backing field compiler sinh. `PropertyInfo.GetCustomAttributes` không thấy attribute đó. Field ẩn có tên dạng `<Name>k__BackingField` và là `private`; tên này không phải contract ổn định giữa compiler. `[property: ...]` trên positional record thì ngược lại: đọc bằng `PropertyInfo`.

### 3.2 Đối số được phép

Đối số attribute phải biểu diễn được trong metadata: các kiểu số nguyên, `bool`, `char`, `float`, `double`, `string`, enum, `Type`, `object` chứa giá trị attribute hợp lệ, hoặc mảng một chiều của những kiểu hợp lệ. Giá trị là constant expression, `typeof(...)` hay mảng attribute hợp lệ; không dùng `decimal`, `DateTime`, object tùy ý hoặc gọi method để tính đối số. Named argument cần public instance field không readonly hoặc public writable instance property. [Đặc tả attributes](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/attributes).

```csharp
[AttributeUsage(AttributeTargets.Class)]
public sealed class PayloadAttribute : Attribute
{
    public PayloadAttribute(Type payloadType) => PayloadType = payloadType;
    public Type PayloadType { get; }
}

[Payload(typeof(Dictionary<,>))] // Unbound generic Type hợp lệ trong đối số.
public sealed class Envelope { }
```

**Generic attribute (C# 11+):** có thể khai báo `PayloadAttribute<T> : Attribute` rồi dùng `[Payload<int>]`. Kiểu sử dụng phải đóng hoàn toàn: không dùng type parameter chưa xác định như `[Payload<T>]` trong declaration generic. Generic attribute và attribute nhận `typeof(T)` là hai cách biểu diễn khác nhau.

## 4. Đọc attribute và kế thừa

Ví dụ tiếp nối `InvoiceService` ở mục 2:

```csharp
using System.Reflection;

MethodInfo save = typeof(InvoiceService).GetMethod(
    nameof(InvoiceService.Save), Type.EmptyTypes)
    ?? throw new MissingMethodException();

foreach (AuditAttribute audit in save.GetCustomAttributes<AuditAttribute>(inherit: true))
    Console.WriteLine($"{audit.Category}: {audit.Enabled}");

foreach (CustomAttributeData metadata in save.GetCustomAttributesData())
{
    Console.WriteLine(metadata.AttributeType.Name);
    foreach (CustomAttributeTypedArgument argument in metadata.ConstructorArguments)
        Console.WriteLine(argument.Value);
}
```

`GetCustomAttributes<T>()` tạo attribute instance và chạy constructor / named setters; giữ constructor nhẹ, không làm I/O. `CustomAttributeData` đọc type và arguments mà không chạy constructor, phù hợp khi chỉ inspect metadata. Nó mô tả attributes gắn trực tiếp, không tự tổng hợp inheritance. [Đọc custom attributes](https://learn.microsoft.com/en-us/dotnet/fundamentals/reflection/accessing-custom-attributes).

**Pitfalls:**

- `GetCustomAttribute<T>()` phù hợp khi tối đa một kết quả; nếu có nhiều attribute hợp lệ, dùng plural API để tránh `AmbiguousMatchException`.
- Class inheritance / method override có thể tham gia truy vấn `inherit: true`; attribute trên interface không tự truyền sang implementing class/method.
- `MemberInfo.GetCustomAttributes(..., inherit)` bỏ qua `inherit` cho property và event. Để tìm theo inheritance chain của chúng, dùng overload thích hợp của `Attribute.GetCustomAttributes`, hoặc generic extension API dựa trên `Attribute`. [API MemberInfo](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.memberinfo.getcustomattributes?view=net-10.0).
- `Attribute.IsDefined` chỉ trả có/không và **không** chạy constructor. Dùng khi attribute không có dữ liệu cần đọc. `IsDefined(..., inherit: true)` với property vẫn đi đường `Attribute`, không cùng bẫy `MemberInfo` ở trên.

```csharp
[AttributeUsage(AttributeTargets.Property, Inherited = true)]
public sealed class NoteAttribute : Attribute
{
    public NoteAttribute(string text) => Text = text;
    public string Text { get; }
}

PropertyInfo name = typeof(Derived).GetProperty(nameof(Derived.Name))!;

// Có thể rỗng dù base property có attribute: MemberInfo bỏ qua inherit.
_ = name.GetCustomAttributes<NoteAttribute>(inherit: true);

NoteAttribute? viaAttribute = Attribute.GetCustomAttribute(
    name, typeof(NoteAttribute), inherit: true) as NoteAttribute;
_ = viaAttribute;

public class BaseDoc
{
    [Note("name")]
    public virtual string Name => "";
}

public sealed class Derived : BaseDoc
{
    public override string Name => "x";
}
```

`AllowMultiple = false`: derived gắn cùng attribute thì kết quả `inherit: true` là bản derived, không phải cả hai. `AllowMultiple = true` thì cả hai có thể xuất hiện. Interface không tham gia: class implement interface không nhận attribute gắn trên member interface, dù `inherit: true`. Đọc attribute của interface bằng `GetInterfaceMap` rồi lấy `InterfaceMethods[i]`.

## 5. Type, member và BindingFlags

```csharp
using System.Reflection;

Type type = typeof(InvoiceService);
MethodInfo[] declaredMethods = type.GetMethods(
    BindingFlags.Public | BindingFlags.Instance | BindingFlags.DeclaredOnly);

foreach (MethodInfo method in declaredMethods)
    Console.WriteLine($"{method.DeclaringType}: {method.Name}");
```

| Flag / API | Quy tắc |
|---|---|
| `Public` / `NonPublic` | Chọn visibility |
| `Instance` / `Static` | Chọn loại member; khi truyền flags phải chọn visibility và loại member cần tìm |
| `DeclaredOnly` | Chỉ member khai báo trên type đang xét |
| `FlattenHierarchy` | Thêm public/protected static members từ base; không kéo private static members của base |
| `IgnoreCase` | Tra tên không phân biệt hoa thường |
| `GetMethod(name, parameterTypes)` | Chọn overload theo signature, tránh chỉ dựa vào tên |
| `GetInterfaces()` / `IsAssignableFrom()` | Kiểm tra interface / khả năng gán kiểu |

`GetMethods()` mặc định trả public instance và static methods, gồm inherited methods theo quy tắc của API. Private member của base thường cần duyệt `BaseType` và dùng `DeclaredOnly` tại mỗi cấp; `FlattenHierarchy` không phải “mọi member của mọi base”. Thứ tự member không nên được dùng làm business contract. [BindingFlags](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.bindingflags?view=net-10.0).

`GetMethod(name)` không thấy member thì trả `null`, không ném. `NonPublic` mới thấy `private` / explicit interface implementation. Tên explicit có dạng `Namespace.IFoo.M`.

```csharp
const BindingFlags any = BindingFlags.Public | BindingFlags.NonPublic
    | BindingFlags.Instance | BindingFlags.Static | BindingFlags.DeclaredOnly;

ConstructorInfo[] ctors = type.GetConstructors(any);
FieldInfo[] fields = type.GetFields(any);
PropertyInfo[] props = type.GetProperties(any);

PropertyInfo? indexer = props.FirstOrDefault(p => p.GetIndexParameters().Length > 0);
MethodInfo? setter = indexer?.GetSetMethod(nonPublic: true);
_ = (ctors, fields, setter);
```

`DeclaringType` là type khai báo member. `ReflectedType` là type mà lần `Get*` này đi từ đó. Method kế thừa lấy qua derived: `DeclaringType` vẫn là base, `ReflectedType` là derived. So sánh `MethodInfo` giữa hai lần lookup khác type dễ fail dù cùng method — so `MetadataToken` + `Module`, hoặc `DeclaringType` + signature.

`GetValue` / `SetValue` trên property gọi accessor, nên chạy validation và có thể ném. Chúng **không** tôn trọng `init` và thường vẫn gán được `readonly` field sau constructor: reflection trong full-trust không phải biên đóng gói. Không thêm lock cho object đích.

Indexer và constructor overload chọn bằng type tham số, giống `GetMethod(name, types)`. `Activator.CreateInstance(type)` dùng public ctor không tham số; không có thì ném. Ctor có tham số: `GetConstructor` + `Invoke`, hoặc `CreateInstance(type, args)` — overload này bind theo runtime type của argument, dễ chọn nhầm khi có `null`.

## 6. Invoke và CreateDelegate

```csharp
using System.Reflection;

MethodInfo contains = typeof(string).GetMethod(
    nameof(string.Contains), [typeof(string)])
    ?? throw new MissingMethodException();

bool found = (bool)contains.Invoke("invoice-42", ["42"])!;

// Open instance delegate: đối số đầu tiên là receiver.
Func<string, string, bool> containsText =
    contains.CreateDelegate<Func<string, string, bool>>();
Console.WriteLine(containsText("invoice-42", "42"));
```

`Invoke` cần target cho instance method; static method dùng `null`. Arguments đi qua `object?[]`, value type có thể bị box, và kiểu kết quả cần cast. Method body ném lỗi thì `Invoke` thông thường bọc trong `TargetInvocationException`; lỗi bind signature/target có thể được ném trực tiếp. Khi cần chuyển tiếp lỗi gốc, đọc `InnerException` và dùng `ExceptionDispatchInfo` để giữ stack trace; xem [exceptions](exceptions.md#63-bảo-toàn-stack-trace-thủ-công-nâng-cao).

Nếu signature đã biết và gọi lặp nhiều lần, cache `MethodInfo` / delegate theo phạm vi phù hợp. `CreateDelegate` kiểm tra compatibility lúc tạo và delegate gọi trực tiếp, không dùng `Invoke` cho mỗi lần gọi. Closed instance delegate giữ receiver; cache lâu có thể giữ object sống lâu. [CreateDelegate](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.methodinfo.createdelegate?view=net-10.0).

```csharp
using System.Runtime.ExceptionServices;

Func<string, bool> closed = contains.CreateDelegate<Func<string, bool>>("invoice-42");
Console.WriteLine(closed("42")); // receiver đã gắn, không truyền lại

MethodInfo parse = typeof(int).GetMethod(nameof(int.Parse), [typeof(string)])!;
try
{
    parse.Invoke(null, ["nope"]); // static: target null. Body ném FormatException
}
catch (TargetInvocationException ex)
{
    ExceptionDispatchInfo.Capture(ex.InnerException!).Throw();
}

try
{
    contains.Invoke("invoice-42", [42]); // argument không khớp signature
}
catch (ArgumentException)
{
    // bind fail — không có InnerException của method body
}
```

Open delegate (`Func<string, string, bool>`) không giữ chuỗi đích. Closed delegate giữ `"invoice-42"` sống bằng delegate. Cache static closed delegate vào instance request là leak. `Invoke` mỗi lần còn box argument value-type và bọc exception; delegate thì không bọc `TargetInvocationException`.

## 7. Generic reflection

### 7.1 Generic type definition và constructed type

```csharp
Type definition = typeof(List<>);
Type constructed = definition.MakeGenericType(typeof(int));

Console.WriteLine(definition.IsGenericTypeDefinition);      // True
Console.WriteLine(constructed.ContainsGenericParameters);   // False
Console.WriteLine(constructed == typeof(List<int>));         // True
Console.WriteLine(constructed.GetGenericTypeDefinition() == definition); // True

var values = (List<int>)(Activator.CreateInstance(constructed)
    ?? throw new InvalidOperationException());
values.Add(42);
```

`GetGenericArguments()` trả type parameters trên definition và type arguments trên constructed type. `ContainsGenericParameters` mới xác định type còn mở hay không; một type đã được construct vẫn có thể chứa parameter chưa đóng. Không tạo instance của open type. `MakeGenericType` cần definition, đúng arity và thỏa runtime constraints. [Generic type reflection](https://learn.microsoft.com/en-us/dotnet/fundamentals/reflection/how-to-examine-and-instantiate-generic-types-with-reflection).

```csharp
Type open = typeof(Dictionary<,>);
Type half = open.MakeGenericType(typeof(string), open.GetGenericArguments()[1]);
Console.WriteLine(half.IsGenericType);               // True — đã MakeGenericType
Console.WriteLine(half.ContainsGenericParameters);    // True — TValue vẫn là parameter
// Activator.CreateInstance(half) ném: type chưa đóng hết.
```

### 7.2 Generic method

```csharp
using System.Reflection;

MethodInfo definition = typeof(GenericHelpers).GetMethod(nameof(GenericHelpers.Echo))
    ?? throw new MissingMethodException();
MethodInfo closed = definition.MakeGenericMethod(typeof(int));

Func<int, int> echoInt = closed.CreateDelegate<Func<int, int>>();
Console.WriteLine(echoInt(42));

public static class GenericHelpers
{
    public static T Echo<T>(T value) => value;
}
```

Generic method definition và generic declaring type là hai tầng riêng. Đóng method trên một declaring type còn mở chưa đủ để invoke. `MakeGenericMethod` có annotation `RequiresDynamicCode` / `RequiresUnreferencedCode`: JIT chạy được một combination không chứng minh Native AOT đã có native code cho combination đó. [MakeGenericMethod](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.methodinfo.makegenericmethod?view=net-10.0).

## 8. Nullable metadata

`typeof(string)` không phân biệt declaration `string` với `string?`. Compiler có thể phát nullable metadata; `NullabilityInfoContext` đọc metadata này cho property, field, event và parameter:

```csharp
using System.Reflection;

PropertyInfo property = typeof(Profile).GetProperty(nameof(Profile.Nickname))!;
NullabilityInfo information = new NullabilityInfoContext().Create(property);
Console.WriteLine(information.ReadState); // Nullable

public sealed class Profile
{
    public string? Nickname { get; set; }
    public List<string?> Tags { get; set; } = [];
}
```

Read/write states có thể khác nhau vì accessor annotations; metadata thiếu có thể cho `Unknown`. Thông tin này không kiểm tra giá trị object lúc chạy. Nullable flow contracts như `[NotNullWhen]`, `[MemberNotNull]` xem [NRT](typesystem.md#101-annotation-attributes); JSON enforcement có giới hạn riêng ở [System.Text.Json](system-text-json.md#4-required-nullable-và-constructor).

Generic và mảng: `ReadState` của `List<string?>` là nullability của **property**, không phải của phần tử. Phần tử nằm ở `GenericTypeArguments` / `ElementType`.

```csharp
PropertyInfo tags = typeof(Profile).GetProperty(nameof(Profile.Tags))!;
NullabilityInfo tagsInfo = new NullabilityInfoContext().Create(tags);
NullabilityInfo element = tagsInfo.GenericTypeArguments[0];
Console.WriteLine(element.ReadState); // Nullable nếu Tags là List<string?>
```

`NullabilityInfoContext` có cache nội bộ không thread-safe; nếu reuse giữa nhiều threads thì đồng bộ truy cập. [NullabilityInfoContext](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.nullabilityinfocontext?view=net-10.0).

## 9. Trimming và Native AOT

### 9.1 Trimming: giữ member có consumer gián tiếp

Trimmer có thể bỏ type/member mà phân tích không thấy được dùng. Reflection từ `typeof(KnownType)` dễ phân tích hơn type name do input quyết định; việc không có compile error chưa đủ bảo đảm lookup thành công sau publish.

```csharp
using System.Diagnostics.CodeAnalysis;
using System.Reflection;

// Caller truyền type đã biết; analyzer có thể giữ các public properties.
string[] names = TypeInspector.PublicPropertyNames(typeof(Profile));

public static class TypeInspector
{
    public static string[] PublicPropertyNames(
        [DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicProperties)]
        Type type)
        => type.GetProperties(BindingFlags.Public | BindingFlags.Instance)
               .Select(property => property.Name)
               .ToArray();
}

```

`DynamicallyAccessedMembers` là yêu cầu trên luồng giá trị `Type` (hoặc type-name string hợp lệ), không phải blanket switch bật mọi reflection. Wrapper tiếp tục nhận `Type` phải truyền tiếp cùng yêu cầu; generic API có thể annotate type parameter. Nếu input thực sự tùy ý, annotation ở method cuối không tự làm caller đáp ứng được yêu cầu. [Xử lý trim warnings](https://learn.microsoft.com/en-us/dotnet/core/deploying/trimming/fixing-warnings).

Chọn **đúng nhóm** member. `All` giữ gần như mọi thứ của type đó và kéo theo dependency — dùng khi API thật sự `GetMembers` không lọc. `PublicMethods` không giữ property, field, hay non-public.

| `DynamicallyAccessedMemberTypes` | Giữ đủ cho |
|---|---|
| `PublicParameterlessConstructor` | `Activator.CreateInstance(type)` ctor công khai không tham số |
| `PublicConstructors` | Mọi public ctor, kể cả có tham số |
| `PublicProperties` | `GetProperties` public instance |
| `PublicMethods` | `GetMethods` public |
| `PublicFields` | Public field |
| `Interfaces` | `GetInterfaces` |
| `NonPublicConstructors` / `NonPublicMethods` / … | Lookup `NonPublic` tương ứng |
| `All` | Hầu hết member; đắt khi type kéo graph lớn |

Thiếu flag đúng nhóm là trim thành công lúc build nhưng `GetProperty` trả `null` lúc chạy. Annotation trên method không giữ type mà caller lấy từ chuỗi tự do.

`[UnconditionalSuppressMessage("Trimming", "IL2075")]` tắt cảnh báo **cả khi** analyzer không chứng minh được an toàn. Chỉ đặt sau khi đã giữ member bằng cách khác (registry, `DynamicDependency` đúng member). `[SuppressMessage]` thường không chặn trimmer; `UnconditionalSuppressMessage` mới là suppress cho IL trim. Suppress không giữ code.

| Annotation | Mục đích | Giới hạn |
|---|---|---|
| `DynamicallyAccessedMembers` | Giữ nhóm member cần thiết cho type đi qua API | Caller phải cung cấp type thỏa yêu cầu; chọn nhóm member phù hợp |
| `DynamicDependency` | Khai báo dependency cụ thể mà phân tích không tự thấy | Dependency chỉ có hiệu lực khi member chứa annotation được giữ; không thay thế flow contract |
| `RequiresUnreferencedCode` | Báo API không bảo đảm tương thích trimming | Chuyển cảnh báo tới caller; không tự giữ member |
| `RequiresDynamicCode` | Báo API có thể cần sinh code lúc chạy | Không làm Native AOT có khả năng JIT |

`DynamicDependency` khai báo dependency từ member chứa annotation tới member/type cần giữ; chọn signature hoặc nhóm member cụ thể. [DynamicDependency](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.codeanalysis.dynamicdependencyattribute?view=net-10.0).

### 9.2 AOT: metadata và native code là hai nhu cầu

Native AOT vẫn hỗ trợ reflection cho các type/member đã được giữ và có code phù hợp. Nó không hỗ trợ nạp plugin IL tùy ý bằng `Assembly.Load*` hay sinh IL bằng `Reflection.Emit`. Generic reflection còn cần native instantiations tương ứng; giữ metadata không bảo đảm mọi combination generic đều chạy được. [Giới hạn Native AOT](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/).

Ưu tiên registry với type đã biết, delegate được tạo sẵn, interface/generics thông thường hoặc source generation khi tập kiểu xác định ở build. `RuntimeFeature.IsDynamicCodeSupported` chỉ cho biết khả năng dynamic code; nó không chứng minh metadata lookup của bạn còn sau trim. Xem [publish và AOT](projects-packages.md#12-native-aot-publishaot--overview--pitfalls) và [JSON source generation](system-text-json.md#7-source-generation).

## 10. Chọn cách triển khai

| Nhu cầu | Cách tiếp cận |
|---|---|
| Gắn thông tin khai báo cho compiler/framework | Attribute đúng target và consumer |
| Inspect attribute mà không chạy constructor | `CustomAttributeData` |
| Gọi method đã biết signature nhiều lần | Delegate; cache với lifetime phù hợp |
| Khám phá kiểu tùy ý lúc chạy trên JIT | Reflection với lookup / overload / exception handling rõ ràng |
| Tập kiểu xác định, cần trimming/AOT | Static registry hoặc source generator; annotate reflection còn lại |
| Mô tả null-state sau helper method | Nullable flow attributes, kèm implementation đúng hợp đồng |

**Pitfall cuối:** `[Obsolete]`, nullable annotations và trim/AOT annotations có consumer khác nhau. Attribute không tự tạo validation runtime, không tự bảo đảm thread safety và không thay đổi lifetime của object.

## 11. Interface, event, field và đối số

### 11.1 Explicit interface và event

Method implement tường minh là `private` và tên gồm tên interface. `GetMethod("M")` public trả `null`.

```csharp
public interface ILog { void Write(string text); }

public sealed class Log : ILog
{
    void ILog.Write(string text) { }
}

Type log = typeof(Log);
InterfaceMapping map = log.GetInterfaceMap(typeof(ILog));
MethodInfo iface = map.InterfaceMethods[0];
MethodInfo impl = map.TargetMethods[0];
impl.Invoke(new Log(), ["hi"]); // impl là method private thật
_ = iface;
```

`GetInterfaceMap` không áp dụng cho interface chính nó. Class implement qua method public cùng tên thì `TargetMethods[i]` là method public đó, không phải bản private.

Event compiler sinh có `add` / `remove`. `EventInfo` không invoke handler. Gọi accessor:

```csharp
public sealed class Widget
{
    public event EventHandler? Clicked;
}

EventInfo clicked = typeof(Widget).GetEvent(nameof(Widget.Clicked))!;
MethodInfo add = clicked.GetAddMethod()!;
var widget = new Widget();
EventHandler handler = static (_, _) => { };
add.Invoke(widget, [handler]);
```

Field không có accessor. `GetValue` / `SetValue` đọc thẳng storage, bỏ qua property. `const` là literal metadata (`FieldInfo.IsLiteral`), không phải storage instance — `GetValue(null)` trên const static. `readonly` vẫn `SetValue` được sau ctor trong runtime thường.

### 11.2 Đối số attribute còn lại

Ngoài các kiểu đã nêu ở mục 3.2, enum và mảng một chiều của kiểu hợp lệ dùng được. `params` trên constructor attribute là đường viết mảng.

```csharp
public enum Level { Low, High }

[AttributeUsage(AttributeTargets.Method, AllowMultiple = false)]
public sealed class GateAttribute : Attribute
{
    public GateAttribute(Level level, params int[] codes)
    {
        Level = level;
        Codes = codes;
    }

    public Level Level { get; }
    public int[] Codes { get; }
}

public static class Api
{
    [Gate(Level.High, 1, 2, 3)]
    public static void Run() { }
}
```

`[Gate(Level.High, new int[] { 1, 2, 3 })]` cùng nghĩa. Named argument đứng **sau** mọi positional. `decimal`, `DateTime`, nullable và lời gọi method không phải đối số attribute — compiler báo ở chỗ gắn, không phải lúc reflection.

`CustomAttributeTypedArgument.Value` với mảng là `ReadOnlyCollection<CustomAttributeTypedArgument>`, không phải `int[]` gốc. Enum có thể hiện là số underlying. Đọc `ArgumentType` trước khi cast.

Default của parameter method không nằm trong attribute. `ParameterInfo.HasDefaultValue` và `DefaultValue` đọc metadata của optional parameter. `DefaultValue` là `DBNull` hoặc `Missing` khi metadata không có literal — kiểm tra `HasDefaultValue` trước. Reflection `Invoke` **không** tự điền default: thiếu argument thì ném, trừ khi bạn tự truyền `Type.Missing` cho optional parameter trên một số overload `Invoke`. Delegate tạo từ `CreateDelegate` cũng không thêm default của reflection; call site C# mới điền default.
