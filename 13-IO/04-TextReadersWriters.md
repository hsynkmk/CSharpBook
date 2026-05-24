# Text Readers and Writers

## What they are

`TextReader` / `TextWriter` are abstractions over **character** (not byte) I/O. They bridge the byte world of `Stream` to the text world of `string` and `char`, handling encoding/decoding transparently.

```
TextReader (abstract)              TextWriter (abstract)
├── StreamReader  (over a Stream)  ├── StreamWriter (over a Stream)
└── StringReader  (over a string)  └── StringWriter (to a StringBuilder)
```

```csharp
using var reader = new StreamReader("file.txt");
string? line = await reader.ReadLineAsync();
string all = await reader.ReadToEndAsync();

using var writer = new StreamWriter("out.txt");
await writer.WriteLineAsync("hello");
```

A `Stream` gives you bytes; a `StreamReader` decodes those bytes into `char`s using an `Encoding`.

---

## `StreamReader`

```csharp
using var reader = new StreamReader(
    stream,
    encoding: Encoding.UTF8,
    detectEncodingFromByteOrderMarks: true,   // honor BOM
    bufferSize: 4096,
    leaveOpen: false);

string? line;
while ((line = await reader.ReadLineAsync()) != null) {
    Process(line);
}
```

### Reading methods

```csharp
string? line = reader.ReadLine();          // one line, null at EOF; strips \r\n
string all = reader.ReadToEnd();           // entire remaining content
int ch = reader.Read();                    // one char, -1 at EOF
int n = reader.Read(buffer, 0, count);     // block of chars
int peeked = reader.Peek();                // look without consuming

// Async versions
await reader.ReadLineAsync();
await reader.ReadToEndAsync(cancellationToken);   // CT overload added .NET 7
```

### Line-by-line for large files

```csharp
// Memory-friendly streaming
using var reader = new StreamReader("huge.log");
string? line;
while ((line = reader.ReadLine()) != null) {
    if (line.Contains("ERROR")) errorCount++;
}
```

For files, `File.ReadLines(path)` wraps this in an `IEnumerable<string>` more conveniently. Use `StreamReader` directly when you have a `Stream` (network, compression, etc.) rather than a file path.

---

## `StreamWriter`

```csharp
using var writer = new StreamWriter("out.txt", append: false, Encoding.UTF8);
writer.WriteLine("line 1");
writer.Write("no newline");
await writer.WriteLineAsync("async line");
await writer.FlushAsync();   // or rely on dispose to flush
```

### AutoFlush

```csharp
var writer = new StreamWriter(stream) { AutoFlush = true };
// Each Write immediately flushes — safer but slower (more syscalls)
```

By default `AutoFlush = false` — writes are buffered and flushed on `Flush()`/dispose. Set `true` for interactive/log scenarios where you want output immediately, at a performance cost.

### NewLine

```csharp
writer.NewLine = "\n";   // force Unix line endings regardless of OS
```

Default is `Environment.NewLine` (`\r\n` on Windows, `\n` on Unix). Set explicitly for cross-platform output formats.

---

## `StringReader` / `StringWriter`

In-memory text I/O — useful for testing, parsing strings line by line, or building text.

```csharp
// Parse a multi-line string
using var sr = new StringReader(multiLineText);
string? line;
while ((line = sr.ReadLine()) != null) { ... }

// Build text (alternative to StringBuilder when an API wants a TextWriter)
using var sw = new StringWriter();
sw.WriteLine("a");
sw.WriteLine("b");
string result = sw.ToString();
```

`StringWriter` wraps a `StringBuilder`. Handy when an API takes a `TextWriter` (e.g., serializers, `Console.SetOut`) and you want to capture output to a string.

---

## Encoding traps

The reader/writer must agree with the file's actual encoding, or you get garbage / `�` replacement characters.

```csharp
// ✓ — explicit UTF-8 without BOM (the modern default)
using var writer = new StreamWriter(path, append: false, new UTF8Encoding(encoderShouldEmitUTF8Identifier: false));

// ⚠ — default StreamWriter(path) historically wrote a UTF-8 BOM; varies by platform/version
```

### BOM (Byte Order Mark)

A BOM is a few bytes at the start of a file indicating encoding (`EF BB BF` for UTF-8). 
- `StreamReader` with `detectEncodingFromByteOrderMarks: true` (default) reads and skips the BOM.
- Writing a BOM can break tools that don't expect it (shell scripts, some JSON parsers).

```csharp
// No BOM (recommended for interop)
var noBom = new UTF8Encoding(false);
using var w = new StreamWriter(path, false, noBom);
```

See [07-Encoding.md](07-Encoding.md) for the full encoding discussion.

---

## `leaveOpen` — don't dispose what you don't own

```csharp
public string ReadFirstLine(Stream stream) {
    // ⚠ — disposes the caller's stream
    using var reader = new StreamReader(stream);
    return reader.ReadLine() ?? "";
}
```

`StreamReader`/`StreamWriter` dispose the underlying stream by default. When the caller owns the stream, pass `leaveOpen: true`:

```csharp
using var reader = new StreamReader(stream, Encoding.UTF8,
    detectEncodingFromByteOrderMarks: true, bufferSize: 1024, leaveOpen: true);
```

---

## `Console` is a TextReader/TextWriter

```csharp
TextWriter stdout = Console.Out;
TextReader stdin = Console.In;
TextWriter stderr = Console.Error;

Console.SetOut(new StreamWriter("log.txt") { AutoFlush = true });   // redirect
Console.WriteLine("goes to file now");
```

This is why you can redirect `Console` output to a file/string — it's just swapping the `TextWriter`. Useful for testing code that writes to `Console`.

---

## Regex with text (brief)

When parsing text you've read, `System.Text.RegularExpressions.Regex` is the workhorse. Prefer the **source-generated** form (C# 11+) for compiled, AOT-safe patterns:

```csharp
public partial class Parser {
    [GeneratedRegex(@"^(?<key>\w+)\s*=\s*(?<value>.+)$")]
    private static partial Regex KeyValue();

    public (string, string)? Parse(string line) {
        var m = KeyValue().Match(line);
        return m.Success ? (m.Groups["key"].Value, m.Groups["value"].Value) : null;
    }
}
```

`[GeneratedRegex]` compiles the pattern at build time — faster than `new Regex(...)` and trimming/AOT-safe. See [Chapter 12 §05](../12-Reflection/05-SourceGenerators.md).

For simple cases, `string` methods (`Split`, `IndexOf`, `StartsWith`, `Span`-based parsing) are faster than regex.

---

## Common bugs

### Forgetting to flush before reading back

```csharp
var writer = new StreamWriter(stream);
writer.WriteLine("data");
stream.Position = 0;
var reader = new StreamReader(stream);
reader.ReadToEnd();   // ⚠ — empty! writer's buffer not flushed yet
```

Call `writer.Flush()` before reading the stream, or dispose the writer first (with `leaveOpen` if you still need the stream).

### Disposing twice / order

When composing, dispose the **outermost** wrapper; it cascades. Disposing inner first can cause "stream closed" errors.

### Wrong encoding → mojibake

Reading a Windows-1252 file as UTF-8 produces `Ã©` instead of `é`. Always know your source encoding; specify it explicitly.

### ReadLine and very long lines

`ReadLine` buffers the whole line in memory. A pathological file with one 2-GB "line" (no `\n`) blows memory. For untrusted input, cap line length or read fixed-size blocks.

---

## Performance notes

- `StreamReader`/`StreamWriter` buffer internally (default 1 KB chars / 4 KB bytes). Increase `bufferSize` for large sequential I/O.
- Async methods (`ReadLineAsync`, `WriteLineAsync`) avoid blocking threads — use in servers.
- For maximum throughput parsing, drop to `Span<char>`/`Utf8` parsing or pipelines rather than `ReadLine` per line.
- `StringBuilder` directly is faster than `StringWriter` when you don't need the `TextWriter` interface.
- Source-generated regex beats `new Regex` and `RegexOptions.Compiled`.

---

## When to use what

| Need | Use |
|---|---|
| Read text file lines | `File.ReadLines` or `StreamReader.ReadLine` |
| Read text over a non-file stream | `StreamReader` |
| Write text file | `StreamWriter` |
| Parse a string line by line | `StringReader` |
| Capture TextWriter output to string | `StringWriter` |
| Redirect Console | `Console.SetOut(TextWriter)` |
| Pattern match text | `[GeneratedRegex]` |

---

## Summary

- `TextReader`/`TextWriter` handle character I/O over byte streams, managing encoding.
- `StreamReader`/`StreamWriter` for files/streams; `StringReader`/`StringWriter` for in-memory text.
- Always specify encoding explicitly; beware the BOM for interop.
- Use `leaveOpen: true` when you don't own the underlying stream.
- Flush (or dispose) writers before reading the stream back.
- `Console.In`/`Out`/`Error` are TextReader/Writer — redirectable.
- Prefer source-generated regex for text parsing.

→ Next: [05-SystemTextJson.md](05-SystemTextJson.md)
