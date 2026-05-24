# Streams

## What a stream is

A `Stream` is an abstraction over a **sequence of bytes** with a position, supporting read, write, and (optionally) seek. It's the universal I/O interface in .NET — files, network sockets, memory, compression, encryption all expose `Stream`.

```csharp
abstract class Stream {
    public abstract bool CanRead { get; }
    public abstract bool CanWrite { get; }
    public abstract bool CanSeek { get; }
    public abstract long Length { get; }
    public abstract long Position { get; set; }

    public abstract int Read(byte[] buffer, int offset, int count);
    public abstract void Write(byte[] buffer, int offset, int count);
    public abstract void Flush();
    // + Span/Memory overloads, async variants, Seek, SetLength...
}
```

Because everything is a `Stream`, you can compose them: read a file → decompress → decrypt → parse, all via stacked streams.

---

## The stream zoo

| Stream | Purpose |
|---|---|
| `FileStream` | File on disk. |
| `MemoryStream` | In-memory byte buffer. |
| `NetworkStream` | TCP socket. |
| `BufferedStream` | Wraps another stream, adds buffering. |
| `GZipStream` / `DeflateStream` / `BrotliStream` | Compression. |
| `CryptoStream` | Encryption/decryption. |
| `StreamReader`/`StreamWriter` | Text over a byte stream (not a Stream themselves — adapters). |
| `Stream.Null` | Discards writes, returns empty reads (like /dev/null). |

---

## `FileStream`

```csharp
using FileStream fs = new(
    path,
    FileMode.OpenOrCreate,
    FileAccess.ReadWrite,
    FileShare.Read,           // others may read while we have it open
    bufferSize: 4096,
    useAsync: true);          // enables true async I/O on supported OSes

byte[] buffer = new byte[1024];
int read = await fs.ReadAsync(buffer);
```

### `FileMode`

| Mode | Behavior |
|---|---|
| `CreateNew` | Create; throw if exists. |
| `Create` | Create or truncate existing. |
| `Open` | Open existing; throw if missing. |
| `OpenOrCreate` | Open existing or create new. |
| `Truncate` | Open existing and set length to 0. |
| `Append` | Open/create, seek to end; write-only. |

### `FileAccess` and `FileShare`

- `FileAccess`: `Read`, `Write`, `ReadWrite` — what *you* will do.
- `FileShare`: `None`, `Read`, `Write`, `ReadWrite`, `Delete` — what *others* may do concurrently.

```csharp
// Allow other readers while you read (common for log tailing)
using var fs = new FileStream(path, FileMode.Open, FileAccess.Read, FileShare.ReadWrite);
```

`FileShare.None` (exclusive) is the cause of "file in use" errors when two processes both want it.

---

## `MemoryStream`

An in-memory, resizable byte buffer that looks like a stream. Used to build/consume byte data without touching disk.

```csharp
using var ms = new MemoryStream();
await JsonSerializer.SerializeAsync(ms, obj);
ms.Position = 0;                          // rewind to read
byte[] bytes = ms.ToArray();              // copy out
// or ms.GetBuffer() returns the internal buffer (may be larger than Length — no copy)
```

`ToArray()` allocates a right-sized copy. `GetBuffer()` returns the backing array (no copy, but length ≠ data length — use `Length`). For high-throughput, prefer `RecyclableMemoryStream` (Microsoft.IO) to pool buffers and avoid LOH allocations.

---

## Reading and writing

### The Read contract (critical)

```csharp
int Read(byte[] buffer, int offset, int count)
```

`Read` returns the number of bytes **actually read**, which may be **less than requested** even if more data is coming. You MUST loop:

```csharp
// ⚠ — wrong: assumes one Read gets everything
int n = stream.Read(buffer, 0, buffer.Length);

// ✓ — read fully
int total = 0;
while (total < count) {
    int n = stream.Read(buffer, total, count - total);
    if (n == 0) break;   // end of stream
    total += n;
}

// ✓ — or use the helper (.NET 7+)
stream.ReadExactly(buffer, 0, count);   // throws if EOF before count
int got = stream.ReadAtLeast(buffer, minBytes, throwOnEndOfStream: false);
```

`Read` returning 0 means **end of stream**. A partial read is normal for network streams especially. This is the #1 stream bug.

### Span/Memory overloads (modern, allocation-friendly)

```csharp
Span<byte> span = stackalloc byte[256];
int read = stream.Read(span);

Memory<byte> mem = buffer.AsMemory(0, 256);
int readAsync = await stream.ReadAsync(mem, cancellationToken);
```

Prefer the `Span`/`Memory` overloads over the `byte[], int, int` ones in new code — cleaner and avoid intermediate arrays.

---

## Copying streams

```csharp
await source.CopyToAsync(destination, bufferSize: 81920, cancellationToken);
```

`CopyTo`/`CopyToAsync` handles the read-loop internally. Default buffer is 81920 bytes (80 KB). Use this instead of hand-rolling the loop.

```csharp
// Compress a file
using var input = File.OpenRead("data.txt");
using var output = File.Create("data.txt.gz");
using var gzip = new GZipStream(output, CompressionLevel.Optimal);
await input.CopyToAsync(gzip);
```

---

## Stream composition (decorator pattern)

Streams wrap streams. Each adds a behavior:

```csharp
// Read an encrypted, gzipped file as text
using var fileStream = File.OpenRead("secret.gz.enc");
using var decrypt   = new CryptoStream(fileStream, decryptor, CryptoStreamMode.Read);
using var decompress = new GZipStream(decrypt, CompressionMode.Decompress);
using var reader     = new StreamReader(decompress, Encoding.UTF8);

string text = await reader.ReadToEndAsync();
```

Bytes flow: disk → decrypt → decompress → decode text. Disposal cascades from outer to inner (`StreamReader` disposes `GZipStream` disposes `CryptoStream` disposes `FileStream`).

---

## Flushing and disposal

```csharp
await stream.FlushAsync();              // flush buffers to the underlying device
stream.Flush(flushToDisk: true);        // FileStream: force OS to write to physical disk
```

- `Flush()` pushes buffered data to the OS.
- `Flush(true)` (FileStream) forces an `fsync` — durable but slow. Use for crash-critical data.
- **Disposing a stream flushes it.** Always `using` your streams.

```csharp
await using var fs = new FileStream(...);   // IAsyncDisposable — flushes async on dispose
```

Use `await using` for async streams so the final flush doesn't block. See [Chapter 08 §08](../08-Concurrency/08-AsyncDisposable.md).

---

## Buffering

Disk and network I/O is slow per-syscall. `BufferedStream` (or `FileStream`'s built-in buffer) batches small reads/writes into larger syscalls.

```csharp
using var raw = new FileStream(path, FileMode.Open);
using var buffered = new BufferedStream(raw, bufferSize: 65536);
// Many small reads now hit the buffer, not the disk each time
```

`FileStream` already buffers (default 4 KB). `BufferedStream` is mainly for wrapping unbuffered streams (e.g., `NetworkStream`) or increasing the buffer size.

Writing many small chunks unbuffered = many syscalls = slow. Buffer them.

---

## `RandomAccess` — offset-based I/O (.NET 6+)

For reading/writing at specific offsets without a stream position (thread-safe, no seeking):

```csharp
using SafeFileHandle handle = File.OpenHandle(path, FileMode.Open, FileAccess.Read);
byte[] buffer = new byte[1024];
long bytesRead = RandomAccess.Read(handle, buffer, fileOffset: 4096);
```

Useful for databases and concurrent readers — multiple threads read different offsets of the same file without contending on a shared `Position`.

---

## Common bugs

### Not looping on Read

Covered above — the cardinal stream sin. Use `ReadExactly`/`CopyTo`/`StreamReader`.

### Forgetting to rewind a MemoryStream

```csharp
JsonSerializer.Serialize(ms, obj);
var bytes = ms.ToArray();   // OK — ToArray reads from 0 regardless of Position
JsonSerializer.Deserialize(ms);   // ⚠ — Position at end, reads nothing. Set ms.Position = 0 first.
```

### Disposing a stream you don't own

```csharp
public void Process(Stream stream) {
    using var reader = new StreamReader(stream);   // ⚠ — disposes the caller's stream!
    ...
}
```

`StreamReader` disposes its underlying stream by default. Use the `leaveOpen: true` constructor parameter when you don't own it:

```csharp
using var reader = new StreamReader(stream, Encoding.UTF8, leaveOpen: true);
```

### Sync over async (blocking)

```csharp
stream.ReadAsync(buffer).Result;   // ⚠ — can deadlock; blocks a thread-pool thread
```

Use `await`. See [Chapter 08 §17](../08-Concurrency/17-CommonAsyncBugs.md).

### Assuming `Length` is cheap or available

`NetworkStream.Length` throws (`NotSupportedException`) — you can't know a socket's length. Check `CanSeek` before using `Length`/`Position`.

---

## Performance notes

- Use `CopyToAsync` with a large buffer (≥64 KB) for bulk transfers.
- Use `Span`/`Memory` overloads to avoid allocations.
- Pool `MemoryStream` buffers with `RecyclableMemoryStream` for high throughput (avoids LOH).
- Set `useAsync: true` on `FileStream` for genuine async file I/O.
- Increase buffer size for high-latency streams; reduce syscall count.
- `RandomAccess` for concurrent offset reads.

---

## When to use what

| Need | Use |
|---|---|
| File bytes | `FileStream` |
| In-memory buffer | `MemoryStream` (pooled for hot paths) |
| Text over bytes | `StreamReader`/`StreamWriter` (§04) |
| Compression | `GZipStream` / `BrotliStream` |
| Bulk copy | `CopyToAsync` |
| Concurrent offset reads | `RandomAccess` + `SafeFileHandle` |
| Very high throughput pipes | `System.IO.Pipelines` (§03) |

---

## Summary

- `Stream` is the byte-sequence abstraction; everything (file, socket, memory, compression) implements it.
- `Read` may return fewer bytes than requested and 0 at EOF — always loop, or use `ReadExactly`/`CopyTo`.
- Prefer `Span`/`Memory` overloads and async methods.
- Compose streams (decorator) for layered transforms.
- Dispose flushes; use `await using`; use `leaveOpen` when you don't own the stream.
- Buffer small I/O; pool buffers for high throughput.

→ Next: [03-Pipelines.md](03-Pipelines.md)
