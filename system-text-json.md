# System.Text.Json

> **Baseline:** .NET **10** / C# **14**. `System.Text.Json` có sẵn trong shared framework. Polymorphism bằng attributes có từ .NET 7; `RespectNullableAnnotations` và `RespectRequiredConstructorParameters` có từ .NET 9.

Chương này tập trung vào JSON contract, cấu hình serializer, custom converters, polymorphism và source generation. Các ví dụ reflection mặc định dành cho ứng dụng JIT; mục source generation chỉ rõ cách dùng metadata được sinh ở build cho trimming/AOT.

## Mục lục

- [1. Serialize và deserialize](#1-serialize-và-deserialize)
- [2. JsonSerializerOptions và defaults](#2-jsonserializeroptions-và-defaults)
- [3. Property, field và enum](#3-property-field-và-enum)
- [4. Required, nullable và constructor](#4-required-nullable-và-constructor)
- [5. Custom converters](#5-custom-converters)
- [6. Polymorphism và discriminator](#6-polymorphism-và-discriminator)
- [7. Source generation](#7-source-generation)
- [8. Trimming và Native AOT](#8-trimming-và-native-aot)
- [9. JsonDocument, JsonElement và JsonNode](#9-jsondocument-jsonelement-và-jsonnode)
- [10. Stream và async](#10-stream-và-async)
- [11. Chọn JSON contract](#11-chọn-json-contract)

---

## 1. Serialize và deserialize

```csharp
using System.Text.Json;

var invoice = new InvoiceDto("INV-42", 125.50m);
string json = JsonSerializer.Serialize(invoice);
InvoiceDto restored = JsonSerializer.Deserialize<InvoiceDto>(json)
    ?? throw new JsonException("Expected an invoice object.");

public sealed record InvoiceDto(string Id, decimal Total);
```

Serializer ánh xạ giữa .NET values và JSON theo một contract; JSON không mang đầy đủ identity hoặc type metadata của CLR. Deserialize về reference type có thể trả null khi payload là JSON `null`, nên caller cần quyết định có chấp nhận hay không. Với dữ liệu chưa biết shape, `Deserialize<object>` thường cho `JsonElement` ở cấu hình mặc định, không suy ra DTO. [Deserialize JSON](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/deserialization).

## 2. JsonSerializerOptions và defaults

### 2.1 Cấu hình tường minh

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    PropertyNameCaseInsensitive = false,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    NumberHandling = JsonNumberHandling.Strict,
    UnmappedMemberHandling = JsonUnmappedMemberHandling.Disallow,
    WriteIndented = true
};

string json = JsonSerializer.Serialize(new InvoiceDto("INV-42", 125.50m), options);
```

| Thiết lập | Tác động |
|---|---|
| `PropertyNamingPolicy` | Tên JSON cho properties; `CamelCase`, `SnakeCaseLower`, v.v. |
| `PropertyNameCaseInsensitive` | So khớp tên khi đọc; không đổi tên lúc ghi |
| `DictionaryKeyPolicy` | Biến đổi string dictionary keys khi ghi; không đổi keys khi đọc |
| `DefaultIgnoreCondition` | `WhenWritingNull` bỏ null; `WhenWritingDefault` còn bỏ `0`, `false`, v.v. khi ghi |
| `NumberHandling` | Mặc định `Strict`; `AllowReadingFromString` nhận số nằm trong JSON string |
| `UnmappedMemberHandling` | Mặc định `Skip`; `Disallow` từ chối properties không thuộc contract (.NET 8+) |
| `AllowDuplicateProperties` | Mặc định cho phép; key trùng thì giá trị **sau** thắng. `false` (.NET 10) ném `JsonException` |
| `ReadCommentHandling` / `AllowTrailingCommas` | Cho phép cú pháp nới lỏng khi đọc nếu được bật |
| `WriteIndented` | Định dạng output; không thay semantics của value |

Chọn `Disallow` khi contract yêu cầu từ chối field lạ; giữ `Skip` khi cần nhận payload mở rộng mà ứng dụng chưa dùng. [Unmapped members](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/missing-members).

Naming policy áp dụng cho tên contract cả khi ghi và đọc; case-insensitive matching là một lựa chọn riêng. `[JsonPropertyName]` có ưu tiên cao hơn policy. [Tên properties và dictionary keys](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/customize-properties).

### 2.2 General và Web

| Cấu hình | Property names khi ghi | Matching khi đọc | Số trong string khi đọc |
|---|---|---|---|
| `new JsonSerializerOptions()` / `Default` | Giữ CLR name | Case-sensitive | Từ chối |
| `new JsonSerializerOptions(JsonSerializerDefaults.Web)` / `Web` | camelCase | Case-insensitive | Chấp nhận |
| `JsonSerializerOptions.Strict` (.NET 10) | Giữ CLR name | Case-sensitive | Từ chối; thêm các cờ dưới |

`JsonSerializerOptions.Web` có từ .NET 9; `Default`, `Web` và `Strict` là instance dùng chung, đã readonly. Khi muốn sửa, tạo options mới từ defaults hoặc clone instance. Cấu hình JSON của ASP.NET Core còn phụ thuộc pipeline MVC / Minimal API và settings ứng dụng. [Options và web defaults](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/configure-options).

`Strict` bật `UnmappedMemberHandling.Disallow`, `AllowDuplicateProperties = false`, `RespectNullableAnnotations` và `RespectRequiredConstructorParameters`. Payload ghi bằng `Default` vẫn đọc được bằng `Strict` nếu shape khớp contract. `Strict` **không** đổi tên sang camelCase — API web vẫn cần `Web` hoặc naming policy riêng, rồi tự bật các cờ bảo vệ nếu cần.

```csharp
string dup = """{ "Value": 1, "Value": -1 }""";
_ = JsonSerializer.Deserialize<Val>(dup)!.Value; // -1 — key sau thắng

JsonSerializer.Deserialize<Val>(dup, JsonSerializerOptions.Strict); // JsonException

sealed record Val(int Value);
```

So khớp trùng tôn trọng naming policy và case-sensitivity: `value` và `Value` là một key khi đang case-insensitive. `JsonDocument.Parse` có `JsonDocumentOptions.AllowDuplicateProperties` riêng, không đọc cờ trên `JsonSerializerOptions`.

### 2.3 Reuse và readonly

Tạo options khi khởi tạo ứng dụng, thêm converters/resolver, rồi reuse. Lần serialize/deserialize đầu tiên khóa options; đổi property hay collection converters sau đó ném `InvalidOperationException`. Metadata cache của options hỗ trợ dùng chung giữa các threads; converter tự viết vẫn cần xử lý shared state phù hợp.

```csharp
// Dành cho JIT/reflection; populate resolver mặc định khi chưa có resolver.
options.MakeReadOnly(populateMissingResolver: true);
Console.WriteLine(options.IsReadOnly); // True
```

`MakeReadOnly()` không đối số yêu cầu đã có `TypeInfoResolver`; overload trên có thể thêm reflection resolver mặc định. Với AOT, đặt generated resolver tường minh rồi dùng `MakeReadOnly()`. [MakeReadOnly](https://learn.microsoft.com/en-us/dotnet/api/system.text.json.jsonserializeroptions.makereadonly?view=net-10.0).

## 3. Property, field và enum

### 3.1 Chọn member của contract

```csharp
using System.Text.Json.Serialization;

public sealed class ProductDto
{
    [JsonPropertyName("product_id")]
    public string Id { get; init; } = "";

    public decimal Price { get; init; }

    [JsonIgnore]
    public string InternalNote { get; set; } = "";

    [JsonInclude]
    public int Revision; // Public field được opt-in.
}
```

Mặc định serialize public properties có getter; deserialize cần setter/init hoặc constructor binding phù hợp. Public fields không tự tham gia: bật `IncludeFields` hoặc dùng `[JsonInclude]` cho từng field. Không dùng `IncludeFields` như cách mở toàn bộ private state. [Fields](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/fields).

`[JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]` cho phép đặt điều kiện riêng theo member. Bỏ null/default lúc ghi làm thay đổi shape của payload; nó không bắt buộc property phải có khi đọc.

`JsonObjectCreationHandling` (.NET 8+): mặc định `Replace` — deserialize **thay** collection/object đang có. `Populate` ghi vào instance sẵn (list không bị thay bằng list mới). Đặt trên property hoặc options. `Populate` trên object có sẵn không phải `JsonConvert.PopulateObject`; vẫn tạo graph từ JSON, chỉ không thay property đã cấu hình populate.

### 3.2 Enum dưới dạng string

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

var enumOptions = new JsonSerializerOptions();
enumOptions.Converters.Add(new JsonStringEnumConverter<InvoiceState>(
    JsonNamingPolicy.CamelCase, allowIntegerValues: false));

string stateJson = JsonSerializer.Serialize(InvoiceState.Paid, enumOptions); // "paid"

public enum InvoiceState { Draft, Paid, Voided }
```

Mặc định enum được ghi bằng số. Generic `JsonStringEnumConverter<TEnum>` có từ .NET 8 và phù hợp với Native AOT hơn factory không generic `JsonStringEnumConverter`, vốn cần dynamic code. `allowIntegerValues: false` từ chối dạng số; string matching khi đọc không phân biệt hoa thường. Đổi tên enum có thể làm thay đổi contract, nên chọn tên wire ổn định. [Generic enum converter](https://learn.microsoft.com/en-us/dotnet/api/system.text.json.serialization.jsonstringenumconverter-1?view=net-10.0).

## 4. Required, nullable và constructor

### 4.1 Thiếu property và explicit null

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

var strictOptions = new JsonSerializerOptions
{
    RespectNullableAnnotations = true,
    RespectRequiredConstructorParameters = true
};

// Lỗi runtime có chủ đích: thiếu Name.
// JsonSerializer.Deserialize<CreateUserDto>("{}", strictOptions);

// Lỗi runtime có chủ đích: Name có mặt nhưng là null.
// JsonSerializer.Deserialize<CreateUserDto>("""{"Name":null}""", strictOptions);

public sealed class CreateUserDto
{
    [JsonRequired]
    public string Name { get; init; } = "";
    public string? Nickname { get; init; }
}
```

| Khai báo / option | Kiểm tra |
|---|---|
| C# `required` hoặc `[JsonRequired]` | Property/field phải xuất hiện trong JSON; thiếu thì `JsonException` |
| `RespectNullableAnnotations = true` | Kiểm tra nullability được hỗ trợ khi đọc và ghi; explicit null trái annotation gây `JsonException` |
| `RespectRequiredConstructorParameters = true` | Non-optional constructor parameters phải có dữ liệu; không buộc optional parameters có default |
| Runtime/domain validation | Giá trị rỗng, range, quan hệ giữa các fields, v.v. |

Required không đồng nghĩa non-null: `[JsonRequired] string?` vẫn cho phép null. Non-null cũng không đồng nghĩa required: riêng `RespectNullableAnnotations` không phát hiện property bị thiếu. Hai flags `Respect*` đều opt-in từ .NET 9. [Required properties và constructor parameters](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/required-properties).

Nullable enforcement không phủ top-level reference value, collection element nullability như `List<string>`, hoặc member dùng generic type parameter. Vì vậy, vẫn kiểm tra kết quả root và invariants cần thiết. NRT ở compile-time và JSON validation là hai consumer khác nhau của annotations. [Giới hạn nullable enforcement](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/nullable-annotations), [nullable flow](typesystem.md#101-annotation-attributes).

### 4.2 Constructor và record

Positional record phù hợp với constructor-based deserialization. Khi có nhiều constructor, `[JsonConstructor]` chọn constructor dùng cho JSON. Parameter phải khớp tên CLR property/field và type, tên parameter so không phân biệt hoa thường; `[JsonPropertyName]` không đổi tên cần match của constructor parameter. Source generation còn cần constructor/member truy cập được từ generated code; tránh dựa vào private access của reflection. [Immutable types và constructors](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/immutability).

```csharp
public sealed record SearchRequest(string Query, int Page = 1);

// Khi RespectRequiredConstructorParameters=true:
// Query bắt buộc; thiếu Page dùng default 1.
```

## 5. Custom converters

### 5.1 Converter cho một kiểu đóng

Ví dụ contract màu RGB là JSON string `"#12ABEF"`, dùng đúng sáu hex digits:

```csharp
using System.Globalization;
using System.Text.Json;
using System.Text.Json.Serialization;

public readonly record struct HexColor(uint Rgb);

public sealed class HexColorConverter : JsonConverter<HexColor>
{
    public override HexColor Read(
        ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
    {
        if (reader.TokenType != JsonTokenType.String)
            throw new JsonException("Expected a #RRGGBB string.");

        string? text = reader.GetString();
        if (text is null || text.Length != 7 || text[0] != '#' ||
            !uint.TryParse(text.AsSpan(1), NumberStyles.AllowHexSpecifier,
                CultureInfo.InvariantCulture, out uint rgb))
            throw new JsonException("Expected six hexadecimal digits after #.");

        return new HexColor(rgb);
    }

    public override void Write(
        Utf8JsonWriter writer, HexColor value, JsonSerializerOptions options)
    {
        if (value.Rgb > 0xFFFFFF)
            throw new JsonException("RGB must fit in 24 bits.");

        writer.WriteStringValue("#" + value.Rgb.ToString("X6", CultureInfo.InvariantCulture));
    }
}
```

Converter đọc primitive không gọi `reader.Read()` để đi sang token kế tiếp; serializer điều phối traversal. Với object/array converter, đọc hết đúng một value và kết thúc tại matching end token. Mọi nhánh cần thống nhất wire format và báo malformed input bằng `JsonException`.

### 5.2 Đăng ký và precedence

```csharp
var colorOptions = new JsonSerializerOptions();
colorOptions.Converters.Add(new HexColorConverter());
string colorJson = JsonSerializer.Serialize(new HexColor(0x12ABEF), colorOptions);
// "#12ABEF"
```

Thứ tự chọn converter: `[JsonConverter]` trên property → converter đầu tiên phù hợp trong `Options.Converters` → `[JsonConverter]` trên type → built-in converter. Reference types/nullable value types thường được serializer xử lý null trước; override `HandleNull` khi converter cần tự xử lý null.

Không gọi lại `JsonSerializer.Serialize/Deserialize` cho chính kiểu đang được converter đó xử lý bằng cùng options: có thể chọn lại converter và recurse. Khi delegate xử lý member khác, dùng contract/type info tương ứng. Factory `JsonConverterFactory` phù hợp với họ kiểu generic trên .NET 10; việc đóng converter bằng `MakeGenericType` / `Activator` cần đánh giá trimming/AOT. [Custom converters](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/converters-how-to), [generic reflection](attributes-reflection.md#7-generic-reflection).

## 6. Polymorphism và discriminator

### 6.1 Contract khai báo trên base type

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

BillingEvent message = new InvoiceIssued("INV-42", 125.50m);
string json = JsonSerializer.Serialize<BillingEvent>(message);
BillingEvent restored = JsonSerializer.Deserialize<BillingEvent>(json)
    ?? throw new JsonException("Expected a billing event.");
// {"kind":"invoice-issued","Total":125.50,"InvoiceId":"INV-42"}
// Thứ tự properties ngoài metadata không phải contract của ví dụ.

[JsonPolymorphic(TypeDiscriminatorPropertyName = "kind")]
[JsonDerivedType(typeof(InvoiceIssued), "invoice-issued")]
[JsonDerivedType(typeof(InvoiceVoided), "invoice-voided")]
public abstract record BillingEvent(string InvoiceId);

public sealed record InvoiceIssued(string InvoiceId, decimal Total)
    : BillingEvent(InvoiceId);
public sealed record InvoiceVoided(string InvoiceId, string Reason)
    : BillingEvent(InvoiceId);
```

Khai báo subtype và discriminator trên base, rồi serialize/deserialize bằng base contract. Serialize trực tiếp bằng derived contract không tự tạo cùng envelope discriminator. Đăng ký subtype không có discriminator có thể mở serialization nhưng không đủ để round-trip về subtype khi đọc base.

Trên .NET 10, runtime không tự tìm mọi derived class trong assembly. Dùng danh sách subtype rõ ràng và discriminator ổn định; tên đó không cần là CLR type name. Unknown discriminator khi đọc và unregistered runtime subtype khi ghi mặc định bị từ chối; các fallback settings có thể làm mất derived data hoặc không áp dụng được nếu base abstract. [Polymorphism](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/polymorphism).

### 6.2 Metadata và source generation

Discriminator mặc định là `$type`; ví dụ dùng `kind` và tránh trùng property name của DTO.

Polymorphism hỗ trợ metadata-based source generation; fast-path serialization không hỗ trợ contract đa hình này. Đăng ký base/derived trong context và giữ metadata mode.

Vòng tham chiếu (`a.Next = a`) mặc định **ném** khi serialize. `ReferenceHandler.Preserve` ghi `$id` / `$ref` để round-trip. Source generator không đọc `ReferenceHandler` trên options lúc chạy cho fast path; từ .NET 10 đặt trên context:

```csharp
[JsonSourceGenerationOptions(ReferenceHandler = JsonKnownReferenceHandler.Preserve)]
[JsonSerializable(typeof(Node))]
internal partial class NodeContext : JsonSerializerContext;

sealed class Node
{
    public Node? Next { get; set; }
}
```

`IgnoreCycles` ghi `null` ở cạnh vòng — mất dữ liệu, không khôi phục graph. `$id` / `$ref` là metadata: mặc định phải đứng đầu object. `AllowOutOfOrderMetadataProperties = true` (.NET 9+) cho phép đứng sau, và deserializer phải buffer cả object.

Các tính năng suy ra `closed` hierarchy / union là .NET 11, không có trên net10.0. Union serialize **không** thêm `$type` — xem [union types](typesystem.md#18-union-types-c-15). `closed` + `[JsonPolymorphic(InferClosedTypePolymorphism = true)]` hoặc `JsonSerializerOptions.InferClosedTypePolymorphism` mới ghi discriminator — xem [OOP](oop.md#26-closed-hierarchies-c-15).

## 7. Source generation

### 7.1 Sinh JsonSerializerContext

Context phải là partial type kế thừa `JsonSerializerContext`. Dùng `InvoiceDto` ở mục 1 và `BillingEvent` / derived types ở mục 6:

```csharp
using System.Text.Json.Serialization;

[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase,
    GenerationMode = JsonSourceGenerationMode.Default,
    RespectNullableAnnotations = true,
    RespectRequiredConstructorParameters = true)]
[JsonSerializable(typeof(InvoiceDto))]
[JsonSerializable(typeof(InvoiceDto[]))]
[JsonSerializable(typeof(BillingEvent))]
[JsonSerializable(typeof(InvoiceIssued))]
[JsonSerializable(typeof(InvoiceVoided))]
internal partial class AppJsonContext : JsonSerializerContext { }
```

```csharp
using System.Text.Json;

var invoice = new InvoiceDto("INV-42", 125.50m);
string json = JsonSerializer.Serialize(invoice, AppJsonContext.Default.InvoiceDto);
InvoiceDto restored = JsonSerializer.Deserialize(json, AppJsonContext.Default.InvoiceDto)
    ?? throw new JsonException("Expected an invoice object.");

BillingEvent message = new InvoiceIssued("INV-42", 125.50m);
string eventJson = JsonSerializer.Serialize(message, AppJsonContext.Default.BillingEvent);
```

Sinh context chưa tự thay mọi call site. Truyền `JsonTypeInfo<T>` như trên, hoặc cấu hình `TypeInfoResolver` bằng context. Với root `InvoiceDto[]`, `List<InvoiceDto>` hoặc kiểu runtime qua `object`, bảo đảm context có contract tương ứng; có metadata của riêng element chưa đủ cho mọi root container.

### 7.2 Modes và resolver

| Mode | Sinh gì | Dùng khi |
|---|---|---|
| `Metadata` | Metadata contract | Deserialize, async/streaming, polymorphism, options không có fast path |
| `Serialization` | Mã ghi được tối ưu | Serialization với contract/options được hỗ trợ; không đủ cho deserialize |
| `Default` | Metadata và serialization code | Có metadata fallback khi fast path không áp dụng |

```csharp
using System.Text.Json.Serialization.Metadata;

var generatedOptions = new JsonSerializerOptions
{
    TypeInfoResolver = AppJsonContext.Default,
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    RespectNullableAnnotations = true,
    RespectRequiredConstructorParameters = true
};
generatedOptions.MakeReadOnly();

JsonTypeInfo<InvoiceDto> invoiceTypeInfo =
    (JsonTypeInfo<InvoiceDto>)generatedOptions.GetTypeInfo(typeof(InvoiceDto));
string json = JsonSerializer.Serialize(new InvoiceDto("INV-42", 125.50m), invoiceTypeInfo);
```

Options trên context mặc định và options truyền riêng cần thống nhất với JSON contract mong muốn. `TypeInfoResolverChain` (.NET 8+) cho phép ghép nhiều contexts; resolver đầu tiên trả metadata cho type được dùng. Fast path không hỗ trợ mọi option/converter; có metadata thì serializer có thể dùng đường metadata thay thế. [Source generation](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/source-generation).

## 8. Trimming và Native AOT

Với app AOT, dùng generated contracts và converters không phụ thuộc runtime code generation. Tắt reflection default giúp phát hiện call site còn dùng reflection ngay trong dev loop:

```xml
<PropertyGroup>
  <JsonSerializerIsReflectionEnabledByDefault>false</JsonSerializerIsReflectionEnabledByDefault>
</PropertyGroup>
```

Setting này áp dụng cả CoreCLR và Native AOT; khi `PublishTrimmed` bật mà không override setting, reflection defaults được tắt tự động. Call site thiếu metadata không tự trở nên tương thích chỉ nhờ attribute trên DTO. Không thêm `DefaultJsonTypeInfoResolver` làm fallback cho AOT để che metadata bị thiếu. [Tắt reflection defaults](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/source-generation#disable-reflection-defaults).

Source generator không tự sửa converter dùng `MakeGenericType`, `Reflection.Emit` hay tùy ý load assembly. Với nhóm kiểu đóng đã biết, đăng ký concrete converters, generated contexts và type info rõ ràng. Xem [trimming/AOT annotations](attributes-reflection.md#9-trimming-và-native-aot) và [cấu hình publish](projects-packages.md#12-native-aot-publishaot--overview--pitfalls).

## 9. JsonDocument, JsonElement và JsonNode

| API | Đặc điểm | Lifetime |
|---|---|---|
| `JsonDocument` | DOM chỉ đọc, parse một JSON value | `IDisposable`; dispose sau khi dùng |
| `JsonElement` | View value trong document; kiểm tra `ValueKind` trước accessor | Thường phụ thuộc document; `Clone()` tạo value độc lập |
| `JsonNode` / `JsonObject` / `JsonArray` | DOM có thể sửa | Managed objects; JSON null có thể biểu diễn bằng null reference |

```csharp
using System.Text.Json;
using System.Text.Json.Nodes;

JsonElement detached;
using (JsonDocument document = JsonDocument.Parse("""{"id":"INV-42"}"""))
{
    detached = document.RootElement.Clone();
}
Console.WriteLine(detached.GetProperty("id").GetString()); // Document đã dispose.

JsonNode node = JsonNode.Parse("""{"status":"draft"}""")
    ?? throw new JsonException("Expected an object.");
node["status"] = "paid";
string changedJson = node.ToJsonString();
```

`GetProperty` dùng tên case-sensitive và ném lỗi nếu thiếu; `TryGetProperty` hữu ích cho field optional. `GetString` / `GetInt32` vẫn cần token đúng loại và số đúng range. Không trả `RootElement` ra ngoài scope đã dispose nếu chưa clone. [JSON DOM](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/use-dom).

Nếu cần round-trip unknown properties cùng DTO, `[JsonExtensionData]` trên `Dictionary<string, JsonElement>` có thể thu chúng thay vì bỏ qua. Chọn việc nhận field lạ hay từ chối bằng contract; tránh vô tình ghi lại dữ liệu chưa được ứng dụng hiểu. [Overflow và extension data](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/handle-overflow).

## 10. Stream và async

```csharp
using System.Text.Json;

// Đặt helper trong type phù hợp; input chứa một JSON array InvoiceDto[].
static async Task<int> CountInvoicesAsync(Stream input, CancellationToken token)
{
    int count = 0;
    await foreach (InvoiceDto? invoice in JsonSerializer.DeserializeAsyncEnumerable(
        input, AppJsonContext.Default.InvoiceDto, cancellationToken: token))
    {
        if (invoice is null)
            throw new JsonException("Array elements must be invoice objects.");
        count++;
    }
    return count;
}
```

`DeserializeAsyncEnumerable<T>` đọc dần elements của JSON array, không materialize toàn bộ `List<T>` trước khi yield. Overload array ở trên không nhận `topLevelValues`; overload có flag này mặc định false, bật true (.NET 9+) để đọc chuỗi top-level JSON values thay vì array. Caller giữ quyền dispose stream; truyền token và xử lý lỗi khi enumeration đang diễn ra. [Streaming API](https://learn.microsoft.com/en-us/dotnet/api/system.text.json.jsonserializer.deserializeasyncenumerable?view=net-10.0), [async streams](async.md).

`SerializeAsync` / `DeserializeAsync` dùng stream và có overload nhận generated type info. Async streaming cần metadata; đừng chỉ sinh mode `Serialization` cho contract cần deserialize/stream. Source generation thay cách cung cấp contract, không quyết định lifetime của stream.

**.NET 10:** `DeserializeAsync` và `DeserializeAsyncEnumerable` nhận `PipeReader` trực tiếp, không cần bọc `Stream`. `SerializeAsync` ghi `IAsyncEnumerable<T>` vào `PipeWriter` thành một JSON array, producer và consumer chạy song song trên cùng pipe.

```csharp
using System.IO.Pipelines;

var pipe = new Pipe();
await JsonSerializer.DeserializeAsync<InvoiceDto>(pipe.Reader, AppJsonContext.Default.InvoiceDto, token);
```

Caller `Complete` writer khi hết byte và `Complete` reader khi đọc xong. Hủy token không dispose pipe.

## 11. Chọn JSON contract

| Tình huống | Lựa chọn |
|---|---|
| DTO shape ổn định | Property names, constructor và options rõ ràng |
| Payload không tin cậy | `Strict`, hoặc tự bật `Disallow` + `AllowDuplicateProperties = false` |
| Phải phân biệt thiếu với null | Required contract kết hợp nullable enforcement được hỗ trợ |
| Domain value có format riêng | `JsonConverter<T>` với Read/Write thống nhất |
| Nhiều subtype trong cùng API | Base contract với danh sách derived types và discriminator ổn định |
| App trim/AOT | Generated context + type info; concrete converters phù hợp AOT |
| Inspect JSON không biết shape | `JsonDocument` / `JsonElement` |
| Chỉnh sửa JSON tree | `JsonNode` |
| Array lớn qua stream | `DeserializeAsyncEnumerable` và cancellation |
| Bytes đã nằm trên `PipeReader` | Overload .NET 10, không copy sang `Stream` |

JSON deserialize không thay domain validation. Chọn DTO và wire contract theo dữ liệu cần trao đổi; thay naming, ignore rules, enum labels hoặc discriminator là thay hành vi bên ngoài của ứng dụng.
