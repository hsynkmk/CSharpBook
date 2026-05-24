# System.Text.Json

## What it is

`System.Text.Json` (STJ) is the built-in, high-performance JSON library introduced in .NET Core 3.0. It replaced Newtonsoft.Json as the default for ASP.NET Core. It's `Span`/UTF-8 based, allocation-light, and supports compile-time **source generation** for AOT/trimming.

```csharp
using System.Text.Json;

record Person(string Name, int Age);

string json = JsonSerializer.Serialize(new Person("Alice", 30));
// {"Name":"Alice","Age":30}

Person? p = JsonSerializer.Deserialize<Person>(json);
```

Three APIs at different levels:
1. **`JsonSerializer`** — object ↔ JSON (most common).
2. **`JsonNode` / `JsonObject` / `JsonArray`** — mutable DOM.
3. **`JsonDocument` / `JsonElement`** — read-only DOM, low-alloc.
4. **`Utf8JsonReader` / `Utf8JsonWriter`** — low-level streaming (fastest).

---

## `JsonSerializer` — the high-level API

```csharp
// Serialize
string json = JsonSerializer.Serialize(obj, options);
byte[] utf8 = JsonSerializer.SerializeToUtf8Bytes(obj);   // skip string allocation

// Deserialize
T? obj = JsonSerializer.Deserialize<T>(json);
T? fromBytes = JsonSerializer.Deserialize<T>(utf8Span);

// Async stream (large payloads)
await JsonSerializer.SerializeAsync(stream, obj);
T? result = await JsonSerializer.DeserializeAsync<T>(stream);
```

### Async streaming for large data

```csharp
// Stream a huge collection without buffering it all
await using var fs = File.Create("big.json");
await JsonSerializer.SerializeAsync(fs, hugeEnumerable);

// Deserialize a stream of items lazily (.NET 6+)
await foreach (var item in JsonSerializer.DeserializeAsyncEnumerable<Item>(stream)) {
    Process(item);   // items yielded as they're parsed — constant memory
}
```

`DeserializeAsyncEnumerable` is key for processing large JSON arrays without loading the whole document.

---

## `JsonSerializerOptions`

```csharp
var options = new JsonSerializerOptions {
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,    // Name → name
    PropertyNameCaseInsensitive = true,                    // accept any case on read
    WriteIndented = true,                                  // pretty-print
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    NumberHandling = JsonNumberHandling.AllowReadingFromString,
    Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping, // don't over-escape non-ASCII
    Converters = { new JsonStringEnumConverter() },         // enums as strings
    ReferenceHandler = ReferenceHandler.IgnoreCycles,       // handle circular refs
};
```

### Reuse options instances

```csharp
// ⚠ — creating new options per call is expensive (rebuilds metadata cache)
JsonSerializer.Serialize(obj, new JsonSerializerOptions { ... });

// ✓ — cache and reuse a single options instance
private static readonly JsonSerializerOptions Options = new() { ... };
JsonSerializer.Serialize(obj, Options);
```

`JsonSerializerOptions` caches type metadata internally. Creating one per call destroys that cache — a common performance bug. Options become **immutable** after first use.

There's also `JsonSerializerOptions.Web` (camelCase, case-insensitive — matches ASP.NET defaults) and `JsonSerializerDefaults.Web`.

---

## Attributes for shaping output

```csharp
public class Product {
    [JsonPropertyName("product_id")]     // custom JSON name
    public int Id { get; set; }

    [JsonIgnore]                          // never serialize
    public string Internal { get; set; } = "";

    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingDefault)]
    public int? Discount { get; set; }

    [JsonPropertyOrder(-1)]               // control ordering
    public string Type { get; set; } = "";

    [JsonInclude]                          // include non-public/field
    internal string Sku = "";

    [JsonExtensionData]                    // capture unknown properties
    public Dictionary<string, JsonElement>? Extra { get; set; }
}
```

`[JsonConstructor]` marks which constructor to use for deserialization. Records work automatically — STJ matches JSON properties to the primary constructor parameters by name.

---

## `JsonNode` — mutable DOM

When you don't have (or want) a type, manipulate JSON as a tree:

```csharp
using System.Text.Json.Nodes;

JsonNode root = JsonNode.Parse(json)!;
string name = root["user"]!["name"]!.GetValue<string>();
root["user"]!["age"] = 31;                       // mutate
root["tags"] = new JsonArray("a", "b");          // add
string updated = root.ToJsonString();

// Build from scratch
var obj = new JsonObject {
    ["id"] = 1,
    ["items"] = new JsonArray(new JsonObject { ["sku"] = "X" })
};
```

Use `JsonNode` for dynamic JSON shapes, partial updates, or when the schema is unknown. It replaces the old `dynamic`-based Newtonsoft `JObject` style with a typed API.

---

## `JsonDocument` / `JsonElement` — read-only, low-alloc

For **read-only** inspection of large JSON with minimal allocation:

```csharp
using JsonDocument doc = JsonDocument.Parse(utf8Bytes);
JsonElement root = doc.RootElement;

string name = root.GetProperty("name").GetString()!;
int age = root.GetProperty("age").GetInt32();

foreach (JsonElement item in root.GetProperty("items").EnumerateArray()) {
    Console.WriteLine(item.GetProperty("id").GetInt32());
}

if (root.TryGetProperty("optional", out var opt)) { ... }
```

`JsonDocument` is `IDisposable` (it rents pooled buffers) — always `using`. `JsonElement` values are only valid while the document is alive; copy out (`.Clone()`) if you need to keep them.

`JsonDocument` (read-only, pooled) is faster and lighter than `JsonNode` (mutable). Use it when you only read.

---

## `Utf8JsonReader` / `Utf8JsonWriter` — streaming low-level

The fastest, lowest-allocation API. Forward-only, manual. Used by custom converters and high-perf parsers.

```csharp
// Writer
var buffer = new ArrayBufferWriter<byte>();
using var writer = new Utf8JsonWriter(buffer);
writer.WriteStartObject();
writer.WriteString("name", "Alice");
writer.WriteNumber("age", 30);
writer.WriteEndObject();
writer.Flush();

// Reader (ref struct — can't be used across await)
var reader = new Utf8JsonReader(utf8Bytes);
while (reader.Read()) {
    if (reader.TokenType == JsonTokenType.PropertyName)
        Console.WriteLine(reader.GetString());
}
```

`Utf8JsonReader` is a `ref struct` — it operates directly on `ReadOnlySpan<byte>` and can't cross `await` boundaries. Use it inside synchronous parse loops.

---

## Custom converters

For types STJ can't handle out of the box (custom date formats, value objects):

```csharp
public class UnixDateTimeConverter : JsonConverter<DateTime> {
    public override DateTime Read(ref Utf8JsonReader reader, Type t, JsonSerializerOptions o) =>
        DateTimeOffset.FromUnixTimeSeconds(reader.GetInt64()).UtcDateTime;

    public override void Write(Utf8JsonWriter writer, DateTime value, JsonSerializerOptions o) =>
        writer.WriteNumberValue(((DateTimeOffset)value).ToUnixTimeSeconds());
}

// Register
var opts = new JsonSerializerOptions { Converters = { new UnixDateTimeConverter() } };
```

Or apply per-property with `[JsonConverter(typeof(UnixDateTimeConverter))]`.

---

## Source generation (AOT / performance)

Reflection-based serialization isn't AOT/trimming-safe and pays startup cost. The **source generator** emits serialization code at compile time:

```csharp
[JsonSourceGenerationOptions(PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase)]
[JsonSerializable(typeof(Person))]
[JsonSerializable(typeof(List<Order>))]
public partial class AppJsonContext : JsonSerializerContext {}

// Usage — no reflection
string json = JsonSerializer.Serialize(person, AppJsonContext.Default.Person);
Person? p = JsonSerializer.Deserialize(json, AppJsonContext.Default.Person);
```

Benefits: faster startup, AOT-compatible, trimming-safe, ~2× faster serialize in some cases. Required for NativeAOT. See [Chapter 12 §05](../12-Reflection/05-SourceGenerators.md).

---

## Polymorphic serialization

```csharp
[JsonPolymorphic(TypeDiscriminatorPropertyName = "$type")]
[JsonDerivedType(typeof(Dog), "dog")]
[JsonDerivedType(typeof(Cat), "cat")]
public abstract class Animal { public string Name { get; set; } = ""; }

public class Dog : Animal { public bool GoodBoy { get; set; } }
public class Cat : Animal { public int Lives { get; set; } }

// Serializes with a "$type":"dog" discriminator; deserializes to the right subclass
string json = JsonSerializer.Serialize<Animal>(new Dog { Name = "Rex", GoodBoy = true });
```

Built-in since .NET 7. Earlier, you needed a custom converter.

---

## STJ vs Newtonsoft.Json

| Aspect | System.Text.Json | Newtonsoft.Json |
|---|---|---|
| Performance | Faster, Span/UTF-8 | Slower |
| Allocations | Lower | Higher |
| AOT/trimming | ✓ (source-gen) | ✗ |
| Default in ASP.NET Core | ✓ | opt-in |
| Flexibility | Good, improving | Very flexible (mature) |
| `dynamic`/`JObject` | `JsonNode` | `JObject` |
| Case-insensitive by default | No (opt-in) | Yes |

STJ is the default choice. Use Newtonsoft only for legacy code or features STJ lacks (e.g., some advanced `JsonConverter` scenarios, `[JsonProperty]` with complex settings).

---

## Common bugs

### New options per call

Covered above — cache `JsonSerializerOptions`.

### Case sensitivity surprises

STJ is **case-sensitive by default** on deserialize. JSON `{"name":...}` won't bind to a `Name` property unless `PropertyNameCaseInsensitive = true` or camelCase policy is set. (ASP.NET's `Web` defaults are case-insensitive.)

### Fields not serialized

By default STJ serializes **public properties only** — not fields. Use `[JsonInclude]` on fields or `IncludeFields = true` in options.

### JsonDocument disposed too early

```csharp
JsonElement Get(string json) {
    using var doc = JsonDocument.Parse(json);
    return doc.RootElement;   // ⚠ — element invalid after doc disposed
}
```

Return `.Clone()` or extract the value before disposing.

### Circular references throw

```csharp
JsonSerializer.Serialize(nodeWithCycle);   // ⚠ — JsonException: cycle detected
```

Set `ReferenceHandler.IgnoreCycles` or `Preserve`.

---

## Performance notes

- `SerializeToUtf8Bytes` / writing to a `Stream` avoids the intermediate `string`.
- `DeserializeAsyncEnumerable` streams large arrays at constant memory.
- `JsonDocument` (read-only, pooled) beats `JsonNode` for read-only access.
- Source generation removes reflection startup cost and is AOT-safe.
- Reuse `JsonSerializerOptions`.
- `Utf8JsonReader`/`Writer` for the absolute hot path.

---

## Summary

- `JsonSerializer` for object ↔ JSON; reuse a cached `JsonSerializerOptions`.
- `JsonNode` (mutable DOM), `JsonDocument`/`JsonElement` (read-only, low-alloc), `Utf8JsonReader/Writer` (streaming low-level).
- Shape output with `[JsonPropertyName]`, `[JsonIgnore]`, naming policies.
- Source generation (`JsonSerializerContext`) for AOT/trimming and faster startup.
- STJ is the default; faster and lower-allocation than Newtonsoft.
- Watch case sensitivity, fields-vs-properties, options reuse, and `JsonDocument` lifetime.

→ Next: [06-XmlAndYaml.md](06-XmlAndYaml.md)
