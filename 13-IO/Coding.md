# Chapter 13 — I/O & Serialization — Coding Problems

---

## Problem 1: Count lines matching a pattern in a huge file

Count lines containing "ERROR" in a multi-gigabyte log without loading it all into memory.

<details><summary>Solution</summary>

```csharp
long CountErrors(string path) =>
    File.ReadLines(path).Count(line => line.Contains("ERROR", StringComparison.Ordinal));
```

`ReadLines` is lazy (`IEnumerable<string>`) — constant memory. `ReadAllLines` would load the whole file. Use `Ordinal` comparison for speed (no culture tables).

Async version (.NET 7+):

```csharp
async Task<long> CountErrorsAsync(string path) {
    long count = 0;
    await foreach (var line in File.ReadLinesAsync(path))
        if (line.Contains("ERROR", StringComparison.Ordinal)) count++;
    return count;
}
```

</details>

---

## Problem 2: Atomic config write

Write a config string to disk so that a crash mid-write never leaves a corrupt file.

<details><summary>Solution</summary>

```csharp
public static void WriteAtomic(string path, string content) {
    string dir = Path.GetDirectoryName(Path.GetFullPath(path))!;
    Directory.CreateDirectory(dir);
    string tmp = Path.Combine(dir, Path.GetRandomFileName());
    try {
        File.WriteAllText(tmp, content, new UTF8Encoding(false));   // no BOM
        File.Move(tmp, path, overwrite: true);                      // atomic rename
    } catch {
        if (File.Exists(tmp)) File.Delete(tmp);                     // cleanup on failure
        throw;
    }
}
```

The rename is atomic within a volume — readers see old-or-complete, never partial. Temp file in the same directory ensures same-volume rename.

</details>

---

## Problem 3: Read fully from a stream

Implement a method that reads exactly `count` bytes from a stream into a buffer, handling partial reads.

<details><summary>Solution</summary>

```csharp
// .NET 7+ — built in:
stream.ReadExactly(buffer, 0, count);   // throws EndOfStreamException if short

// Manual implementation (the contract every stream reader must respect):
public static void ReadExactly(Stream s, byte[] buffer, int offset, int count) {
    int total = 0;
    while (total < count) {
        int read = s.Read(buffer, offset + total, count - total);
        if (read == 0) throw new EndOfStreamException();
        total += read;
    }
}
```

The loop is mandatory — a single `Read` may return fewer bytes than requested (especially on network streams).

</details>

---

## Problem 4: Gzip-compress a file

Compress `input.txt` to `input.txt.gz` using stream composition, then decompress to verify.

<details><summary>Solution</summary>

```csharp
using System.IO.Compression;

async Task CompressAsync(string src, string dst) {
    await using var input = File.OpenRead(src);
    await using var output = File.Create(dst);
    await using var gzip = new GZipStream(output, CompressionLevel.Optimal);
    await input.CopyToAsync(gzip);
}

async Task DecompressAsync(string src, string dst) {
    await using var input = File.OpenRead(src);
    await using var gzip = new GZipStream(input, CompressionMode.Decompress);
    await using var output = File.Create(dst);
    await gzip.CopyToAsync(output);
}
```

Stream composition: file → gzip → file. `CopyToAsync` handles the read-loop. `await using` flushes asynchronously on dispose. Disposal cascades outer→inner.

</details>

---

## Problem 5: Serialize/deserialize with System.Text.Json

Round-trip a record to JSON with camelCase naming and enum-as-string.

<details><summary>Solution</summary>

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

enum Status { Active, Suspended }
record User(int Id, string Name, Status Status);

private static readonly JsonSerializerOptions Options = new() {
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    Converters = { new JsonStringEnumConverter() },
    WriteIndented = true,
};

string json = JsonSerializer.Serialize(new User(1, "Alice", Status.Active), Options);
// { "id": 1, "name": "Alice", "status": "Active" }
User? back = JsonSerializer.Deserialize<User>(json, Options);
```

Cache `Options` as static — never create per call. Records deserialize via their primary constructor automatically.

</details>

---

## Problem 6: Stream a large JSON array without buffering

A 5-GB JSON file contains an array of millions of records. Process each at constant memory.

<details><summary>Solution</summary>

```csharp
async Task ProcessLargeJsonAsync(string path) {
    await using var stream = File.OpenRead(path);
    await foreach (var record in JsonSerializer.DeserializeAsyncEnumerable<Record>(stream)) {
        Process(record);   // records yielded as parsed — never holds the whole array
    }
}
```

`DeserializeAsyncEnumerable<T>` parses the array lazily, yielding items as they're read. Memory stays constant regardless of file size — the key API for big JSON. Contrast with `Deserialize<List<Record>>` which loads everything.

</details>

---

## Problem 7: Custom JSON converter

Write a converter that serializes `DateTime` as a Unix timestamp (seconds).

<details><summary>Solution</summary>

```csharp
public class UnixTimeConverter : JsonConverter<DateTime> {
    public override DateTime Read(ref Utf8JsonReader reader, Type t, JsonSerializerOptions o) =>
        DateTimeOffset.FromUnixTimeSeconds(reader.GetInt64()).UtcDateTime;

    public override void Write(Utf8JsonWriter writer, DateTime value, JsonSerializerOptions o) =>
        writer.WriteNumberValue(new DateTimeOffset(value, TimeSpan.Zero).ToUnixTimeSeconds());
}

// Register globally
var opts = new JsonSerializerOptions { Converters = { new UnixTimeConverter() } };

// Or per-property:
public class Event {
    [JsonConverter(typeof(UnixTimeConverter))]
    public DateTime Timestamp { get; set; }
}
```

`Read` uses `Utf8JsonReader` (a ref struct) directly. `Write` uses `Utf8JsonWriter`. This is how you handle non-standard formats.

</details>

---

## Problem 8: Query JSON with JsonNode (unknown schema)

Given arbitrary JSON, extract `user.profile.email` if present, else return null, without defining types.

<details><summary>Solution</summary>

```csharp
using System.Text.Json.Nodes;

string? GetEmail(string json) {
    JsonNode? root = JsonNode.Parse(json);
    return root?["user"]?["profile"]?["email"]?.GetValue<string>();
}
```

`JsonNode` indexers return null for missing keys; the `?.` chain short-circuits safely. For read-only access of large docs, `JsonDocument` is lighter:

```csharp
string? GetEmailDoc(string json) {
    using var doc = JsonDocument.Parse(json);
    if (doc.RootElement.TryGetProperty("user", out var u) &&
        u.TryGetProperty("profile", out var p) &&
        p.TryGetProperty("email", out var e))
        return e.GetString();
    return null;
}
```

</details>

---

## Problem 9: Parse and build XML with namespaces

Parse an RSS-like document in a namespace and extract item titles.

<details><summary>Solution</summary>

```csharp
using System.Xml.Linq;

IEnumerable<string> GetTitles(string xml) {
    XNamespace ns = "http://example.com/feed";
    XDocument doc = XDocument.Parse(xml);
    return doc.Descendants(ns + "item")
              .Select(item => (string)item.Element(ns + "title")!)
              .ToList();
}
```

The `ns + "item"` is essential — without the namespace, `Descendants("item")` returns nothing. This is the most common LINQ-to-XML bug.

</details>

---

## Problem 10: The culture-aware number bug

This code corrupts data when run on a German machine. Fix it.

```csharp
void Save(double value) => File.WriteAllText("v.txt", value.ToString());
double Load() => double.Parse(File.ReadAllText("v.txt"));
```

<details><summary>Solution</summary>

```csharp
void Save(double value) =>
    File.WriteAllText("v.txt", value.ToString(CultureInfo.InvariantCulture));

double Load() =>
    double.Parse(File.ReadAllText("v.txt"), CultureInfo.InvariantCulture);
```

On a German machine, `1234.56.ToString()` produces `"1234,56"` (comma decimal). Parsed on a US machine, the comma is read as a thousands separator → `123456`. Always use `InvariantCulture` for persistence. Use the user's culture only for display.

</details>

---

## Problem 11: Safe path join (prevent traversal)

A web handler joins a base directory with a user-supplied filename. Prevent `../` escaping the base.

<details><summary>Solution</summary>

```csharp
public static string SafeCombine(string baseDir, string userPath) {
    string baseFull = Path.GetFullPath(baseDir);
    string combined = Path.GetFullPath(Path.Combine(baseFull, userPath));

    // Ensure the resolved path is still under baseDir
    if (!combined.StartsWith(baseFull + Path.DirectorySeparatorChar, StringComparison.Ordinal)
        && combined != baseFull)
        throw new UnauthorizedAccessException($"Path escapes base: {userPath}");

    return combined;
}

// SafeCombine("/var/data", "report.txt")        → /var/data/report.txt ✓
// SafeCombine("/var/data", "../../etc/passwd")   → throws ✗
```

`GetFullPath` resolves `..` segments; comparing against the base prefix detects escapes. Critical for any user-supplied path component.

</details>

---

## Problem 12: AOT-safe serialization with source generation

Set up source-generated JSON for a type so it works under NativeAOT.

<details><summary>Solution</summary>

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

public record Order(int Id, string Customer, decimal Total, List<string> Items);

[JsonSourceGenerationOptions(PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase)]
[JsonSerializable(typeof(Order))]
[JsonSerializable(typeof(List<Order>))]
public partial class AppJsonContext : JsonSerializerContext {}

// Usage — no reflection, AOT-safe, faster startup
string json = JsonSerializer.Serialize(order, AppJsonContext.Default.Order);
Order? back = JsonSerializer.Deserialize(json, AppJsonContext.Default.Order);

// For a list:
string listJson = JsonSerializer.Serialize(orders, AppJsonContext.Default.ListOrder);
```

The generator emits serialization code at compile time. Reflection-based serialization throws under NativeAOT and pays startup cost; this is the production pattern for AOT/trimmed apps. The `JsonSerializerContext` partial class is completed by the source generator.

</details>

---

## Problem 13: Tee a stream (write while reading)

Read from a source stream while simultaneously writing a copy to a backup, in one pass.

<details><summary>Solution</summary>

```csharp
async Task<byte[]> ReadAndBackupAsync(Stream source, string backupPath) {
    await using var backup = File.Create(backupPath);
    using var memory = new MemoryStream();

    byte[] buffer = ArrayPool<byte>.Shared.Rent(81920);
    try {
        int read;
        while ((read = await source.ReadAsync(buffer)) > 0) {
            await backup.WriteAsync(buffer.AsMemory(0, read));   // copy to backup
            await memory.WriteAsync(buffer.AsMemory(0, read));   // collect for return
        }
    } finally {
        ArrayPool<byte>.Shared.Return(buffer);
    }
    return memory.ToArray();
}
```

Single pass through the source, writing to two destinations. `ArrayPool` avoids allocating a buffer per call. `buffer.AsMemory(0, read)` writes only the valid portion. Demonstrates the read-loop, Memory overloads, and pooling together.

</details>

---

These problems cover the I/O essentials: lazy reading, atomic writes, the stream Read contract, composition, modern JSON (streaming + source-gen), XML namespaces, and the culture-formatting trap.

→ Back to [Chapter 13 README](README.md). Next chapter: [Chapter 14 — Interop & AOT](../14-InteropAOT/README.md).
