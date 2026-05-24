# UTF-8 String Literals (C# 11+)

## What it is

C# 11 added the `u8` suffix on string literals — instead of producing a `string` (UTF-16), the literal produces a **`ReadOnlySpan<byte>` of UTF-8 encoded bytes**. No allocation, no encoding step at runtime.

```csharp
ReadOnlySpan<byte> http = "HTTP/1.1 200 OK\r\n"u8;
ReadOnlySpan<byte> contentType = "application/json"u8;

// Comparison without allocation
bool isHttp = data.StartsWith("HTTP/1.1"u8);

// Writing to a stream
stream.Write("hello world"u8);
```

Crucial for network code, parsers, and any high-throughput byte-oriented work — eliminates UTF-16 → UTF-8 conversions on the hot path.

---

## Why it exists

Network protocols, file formats, and modern web APIs use UTF-8. .NET strings are UTF-16. Every comparison, every write, requires conversion:

```csharp
// Pre-u8
bool isHttp = data.SequenceEqual(Encoding.UTF8.GetBytes("HTTP/1.1"));   // allocates per call!
```

`Encoding.UTF8.GetBytes("HTTP/1.1")` allocates a `byte[]` each time. For 1M comparisons, 1M allocations.

With `u8`:

```csharp
bool isHttp = data.SequenceEqual("HTTP/1.1"u8);   // ReadOnlySpan<byte> — compile-time bytes
```

The bytes are embedded in the assembly. Zero allocation. Faster comparison.

For HTTP servers (Kestrel), JSON parsers (System.Text.Json), serializers — this is the difference between MB/s and GB/s.

---

## Syntax

```csharp
ReadOnlySpan<byte> bytes = "hello"u8;
```

Add `u8` after the closing quote. The literal is a UTF-8 encoded byte sequence.

For raw strings:

```csharp
ReadOnlySpan<byte> json = """{"name":"Alice"}"""u8;   // raw + u8
```

The `u8` applies to the whole literal.

For multi-byte chars:

```csharp
ReadOnlySpan<byte> emoji = "😀"u8;   // 4 bytes (UTF-8 encoding of U+1F600)
ReadOnlySpan<byte> chinese = "你好"u8;   // 6 bytes (3 per char in UTF-8)
```

Works for any Unicode content.

---

## What's the type?

`"..."u8` is `ReadOnlySpan<byte>`. NOT `byte[]`, NOT `string`.

```csharp
ReadOnlySpan<byte> ok = "x"u8;
byte[] arr = "x"u8;             // ⚠ compile error — can't implicitly convert
byte[] arr2 = "x"u8.ToArray();   // ✓ but allocates
```

If you need a byte array:

```csharp
byte[] arr = "x"u8.ToArray();   // copies the bytes to a new array
```

For most byte-comparison work, ReadOnlySpan<byte> is what you want — Span APIs accept it directly.

---

## Where the bytes live

The UTF-8 bytes are embedded in the assembly's **read-only data section**. No heap allocation, no copy. The `ReadOnlySpan<byte>` is just a pointer into that data + length.

For `"hello"u8`:
- Assembly has 5 bytes of read-only data: `h`, `e`, `l`, `l`, `o` (or 0x68, 0x65, 0x6C, 0x6C, 0x6F).
- At runtime, the literal evaluates to a Span pointing at those bytes.
- No allocation. Ever.

Lookups in switch statements, comparison loops, prefix checks — all allocation-free.

---

## Common patterns

### Protocol parsing

```csharp
public static bool IsHttpRequest(ReadOnlySpan<byte> data) {
    return data.StartsWith("GET "u8) ||
           data.StartsWith("POST "u8) ||
           data.StartsWith("PUT "u8) ||
           data.StartsWith("DELETE "u8);
}
```

Each comparison is a fast byte sequence check, no allocation.

### JSON tokens

```csharp
ReadOnlySpan<byte> nullToken = "null"u8;
ReadOnlySpan<byte> trueToken = "true"u8;
ReadOnlySpan<byte> falseToken = "false"u8;

if (token.SequenceEqual(nullToken)) { /* ... */ }
```

System.Text.Json uses this internally.

### Header writing

```csharp
public static void WriteHttpResponse(Stream s, int status) {
    s.Write("HTTP/1.1 "u8);
    s.Write(status.ToString().AsSpan());   // status needs conversion
    s.Write(" OK\r\n"u8);
    s.Write("Content-Type: text/plain\r\n"u8);
    s.Write("\r\n"u8);
}
```

Header bytes are baked into the assembly. Each Write is a span copy, no encoding work.

### Content-type matching

```csharp
private static readonly Dictionary<string, ReadOnlyMemory<byte>> ContentTypes = new() {
    { "json", "application/json"u8.ToArray() },
    { "html", "text/html"u8.ToArray() },
    { "css", "text/css"u8.ToArray() },
};
```

`ToArray()` is needed only because Dictionary values can't be `ReadOnlySpan<byte>` (not a regular type). For local use, just keep as Span.

---

## Combining with raw strings

```csharp
ReadOnlySpan<byte> json = """
    {
        "name": "Alice",
        "age": 30
    }
    """u8;
```

Embedded multi-line UTF-8 — JSON test fixtures, HTTP body templates, etc.

---

## Operations on UTF-8 spans

```csharp
ReadOnlySpan<byte> data = "hello world"u8;
data.Length;                      // 11
data.IndexOf((byte)' ');          // 5
data[..5].SequenceEqual("hello"u8); // true
data.Contains("world"u8);          // true (.NET 7+ contains span overload)
```

`MemoryExtensions` provides span-on-span operations (IndexOf, StartsWith, EndsWith, SequenceEqual, etc.).

---

## Converting between string and UTF-8 span

```csharp
// string → UTF-8 bytes (allocates)
byte[] bytes = Encoding.UTF8.GetBytes("hello");

// UTF-8 span → string (allocates)
string s = Encoding.UTF8.GetString(data);

// Or
string s2 = Encoding.UTF8.GetString(data.ToArray());   // double allocation; avoid

// For known-ASCII content:
string ascii = System.Text.Encoding.ASCII.GetString(data);
```

For sustained UTF-8 work, stay in spans. Only convert to string at API boundaries.

---

## Performance comparison

For 1M comparisons against the literal "HTTP/1.1":

**Pre-u8** (allocates each call):
```csharp
bool isHttp = data.SequenceEqual(Encoding.UTF8.GetBytes("HTTP/1.1"));
```
- ~10M allocations (each `GetBytes` allocates a small byte[]).
- ~50-100 ms total (mostly GC pressure).

**With u8**:
```csharp
bool isHttp = data.SequenceEqual("HTTP/1.1"u8);
```
- 0 allocations.
- ~5-10 ms total (pure comparison).

10× speedup, infinite reduction in GC pressure. This is why Kestrel, gRPC, JSON parsers love `u8`.

---

## Internals — how the compiler emits

For `"hello"u8`:

The compiler emits the bytes into a **PrivateImplementationDetails** class as a `readonly struct` of bytes:

```il
.field private static initonly valuetype <PrivateImplementationDetails>/'__StaticArrayInitTypeSize=5'
    '04C4814...'
```

(A constant array of 5 bytes.)

The `"hello"u8` expression compiles to:

```il
ldsflda valuetype <PrivateImplementationDetails>/...
ldc.i4.5
newobj ReadOnlySpan<byte>(byte*, int)
```

Loads the address of the embedded bytes + length, constructs a ReadOnlySpan. Single CPU operation.

No allocation, no copy. The bytes are in the assembly's read-only segment (mapped from disk; shared across processes).

---

## When to use

✓ Protocol parsing (HTTP, gRPC, custom).
✓ JSON / YAML / TOML token comparisons.
✓ Content-type / charset / encoding name checks.
✓ Writing fixed bytes to streams.
✓ Any "compare against a literal byte sequence" hot path.

✗ String content needing manipulation (use `string`).
✗ Localized content (UTF-16 string handling is easier).
✗ Anywhere the API requires `string` (you'd convert anyway).

For "UTF-8 byte hot path," `u8` is the right tool. For general text, stay with `string`.

---

## Common bugs

### Trying to do string ops on `"x"u8`

```csharp
var s = "hello"u8;
s.Length;             // 5 (byte length)
s[0];                  // 104 (byte value of 'h'), not 'h'
s.ToUpperInvariant();  // ⚠ — ReadOnlySpan<byte> doesn't have this
```

It's a byte span, not a string. Operations are byte-oriented. For text-style ops, convert to string first.

### Multi-byte char surprises

```csharp
var s = "café"u8;
s.Length;   // 5 — UTF-8 of "café" is c(1) a(1) f(1) é(2) = 5 bytes
```

`s.Length` is byte length, not character length. For character counts, you'd convert to string or use `Encoding.UTF8.GetCharCount(s)`.

### Storing in fields

```csharp
public class C {
    private ReadOnlySpan<byte> _data = "x"u8;   // ❌ — Span can't be a field
}
```

`ReadOnlySpan<byte>` is a `ref struct` — can't be a class field. Workarounds:
- Use `byte[] _data = "x"u8.ToArray();` (allocates once at field init).
- Use `ReadOnlyMemory<byte> _data` (need different type — but converting from u8 means copying).
- Use a static getter / property:
  ```csharp
  private static ReadOnlySpan<byte> Data => "x"u8;
  ```
  This is the canonical pattern — the literal is embedded; the getter just returns a fresh span each call (zero allocation).

### Comparing across encodings

```csharp
data.SequenceEqual("Hello"u8);
// Won't match data containing "hello" or "HELLO"
```

`u8` is byte-exact. For case-insensitive UTF-8 comparison, you'd need to lowercase first (or define separately).

---

## Performance

- Compile-time: bytes embedded in assembly's read-only section. Zero per-call cost.
- Runtime: span construction is ~1 ns.
- Comparison: SIMD-optimized for spans (vectorized byte compare).
- Memory: 1 copy of the bytes per literal, shared across all uses in the process.

For ASCII-only content of N bytes, that's N bytes in the assembly. Negligible.

---

## `IUtf8SpanFormattable` (.NET 8+)

Beyond literals, .NET 8 added the `IUtf8SpanFormattable` interface for types that can write themselves directly as UTF-8:

```csharp
public bool TryFormat(Span<byte> utf8Destination, out int bytesWritten, ReadOnlySpan<char> format, IFormatProvider? provider) {
    // ... format directly to UTF-8 ...
}
```

Implemented by `int`, `double`, `DateTime`, `Guid`, etc. Combined with `u8` literals, you can build UTF-8 output without ever materializing a `string`:

```csharp
Span<byte> buf = stackalloc byte[64];
int pos = 0;
"User: "u8.CopyTo(buf);
pos += 6;
id.TryFormat(buf[pos..], out int w, default, null);
pos += w;
"\r\n"u8.CopyTo(buf[pos..]);
pos += 2;

await stream.WriteAsync(buf[..pos].ToArray());
```

Combined with raw strings + u8, modern .NET makes UTF-8 string work feel natural.

---

## Summary

- `"hello"u8` — UTF-8 string literal. Type: `ReadOnlySpan<byte>`.
- Bytes are embedded in the assembly. Zero allocation at runtime.
- Use for protocol parsing, JSON tokens, header writing, content-type checks.
- For storage: use a static getter property returning the span (`static ReadOnlySpan<byte> X => "..."u8;`).
- Combine with raw strings (`"""..."""u8`) for multi-line UTF-8 fixtures.
- Massive win for byte-oriented hot paths.

→ Next: [11-InterpolatedStringHandlers.md](11-InterpolatedStringHandlers.md)
