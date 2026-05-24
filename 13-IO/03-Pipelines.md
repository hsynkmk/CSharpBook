# System.IO.Pipelines

## What it is

`System.IO.Pipelines` is a high-performance I/O library designed for **parsing streaming data with minimal allocations and minimal copying**. It solves problems that are painful with raw `Stream`: buffer management, partial reads, and back-pressure.

```csharp
using System.IO.Pipelines;

var pipe = new Pipe();
PipeWriter writer = pipe.Writer;   // producer fills buffers
PipeReader reader = pipe.Reader;   // consumer parses buffers
```

It powers Kestrel (ASP.NET Core's web server), SignalR, and gRPC. You reach for it when writing **network protocol parsers** that must be fast and allocation-light.

---

## Why it exists — the problems with raw Stream parsing

Parsing a protocol from a `Stream` forces you to manage:

1. **Buffer allocation** — how big a `byte[]`? Too small → many reads; too big → wasted memory.
2. **Partial messages** — a `Read` may return half a message. You must buffer the remainder and prepend it to the next read.
3. **Buffer copying** — shuffling leftover bytes to the front of the buffer (memmove) on every read.
4. **Back-pressure** — if the consumer is slow, the producer should slow down (don't read faster than you can process).

Pipelines handles all four. The producer writes into pooled memory; the consumer reads a `ReadOnlySequence<byte>` (possibly spanning multiple non-contiguous buffers) and tells the pipe how much it consumed vs examined.

---

## The core loop

### Consumer (PipeReader)

```csharp
async Task ReadPipeAsync(PipeReader reader) {
    while (true) {
        ReadResult result = await reader.ReadAsync();
        ReadOnlySequence<byte> buffer = result.Buffer;

        SequencePosition? position;
        do {
            // Find a line terminator
            position = buffer.PositionOf((byte)'\n');
            if (position != null) {
                ProcessLine(buffer.Slice(0, position.Value));
                // Skip past the \n
                buffer = buffer.Slice(buffer.GetPosition(1, position.Value));
            }
        } while (position != null);

        // Tell the pipe: we consumed up to `buffer.Start`, examined up to `buffer.End`
        reader.AdvanceTo(buffer.Start, buffer.End);

        if (result.IsCompleted) break;
    }
    await reader.CompleteAsync();
}
```

The key call is **`AdvanceTo(consumed, examined)`**:
- **consumed** — bytes you've fully processed (the pipe can recycle them).
- **examined** — bytes you've looked at (so the pipe knows not to return the same data without more arriving).

If you examined everything but consumed only part (incomplete message), the next `ReadAsync` waits for *more* data before returning — no busy loop.

### Producer (PipeWriter)

```csharp
async Task WritePipeAsync(Socket socket, PipeWriter writer) {
    while (true) {
        Memory<byte> memory = writer.GetMemory(sizeHint: 512);   // pooled buffer
        int bytesRead = await socket.ReceiveAsync(memory, SocketFlags.None);
        if (bytesRead == 0) break;
        writer.Advance(bytesRead);                                // commit written bytes

        FlushResult result = await writer.FlushAsync();           // make available to reader; back-pressure here
        if (result.IsCompleted) break;
    }
    await writer.CompleteAsync();
}
```

`FlushAsync` returns a task that completes when the reader has drained enough — **automatic back-pressure**.

---

## `ReadOnlySequence<byte>` — the multi-segment buffer

Unlike a `byte[]` or `Span<byte>` (contiguous), `ReadOnlySequence<byte>` can span **multiple discontiguous memory segments**. This is what lets the pipe avoid copying leftover bytes — it just chains a new segment.

```csharp
void Process(ReadOnlySequence<byte> seq) {
    if (seq.IsSingleSegment) {
        ParseSpan(seq.FirstSpan);    // fast path: contiguous
    } else {
        foreach (var segment in seq) {    // iterate segments
            ParseSpan(segment.Span);
        }
        // or copy to a contiguous buffer if the parser needs it:
        // seq.CopyTo(contiguousBuffer);
    }
}
```

`SequenceReader<byte>` (a ref struct) helps parse across segments:

```csharp
var sr = new SequenceReader<byte>(buffer);
if (sr.TryReadTo(out ReadOnlySequence<byte> line, (byte)'\n')) {
    ProcessLine(line);
}
```

---

## Connecting a Stream to a Pipe

`Stream` interop is built in:

```csharp
// Read from a stream via a pipe
PipeReader reader = PipeReader.Create(stream);

// Write to a stream via a pipe
PipeWriter writer = PipeWriter.Create(stream);

// Copy a pipe to a stream
await reader.CopyToAsync(stream);
```

This lets you use pipeline parsing on top of any existing `Stream`.

---

## Back-pressure configuration

```csharp
var options = new PipeOptions(
    pauseWriterThreshold: 64 * 1024,   // writer pauses when 64 KB unconsumed
    resumeWriterThreshold: 32 * 1024,  // resumes when drained below 32 KB
    minimumSegmentSize: 4096);
var pipe = new Pipe(options);
```

When unconsumed data exceeds `pauseWriterThreshold`, `FlushAsync` doesn't complete until the reader drains below `resumeWriterThreshold`. This bounds memory and prevents a fast producer from overwhelming a slow consumer.

---

## Pipelines vs Streams

| Aspect | Stream | Pipelines |
|---|---|---|
| Buffer management | You allocate `byte[]` | Pipe pools buffers |
| Partial messages | You re-buffer + copy | `AdvanceTo(consumed, examined)` |
| Memory layout | Contiguous | `ReadOnlySequence` (multi-segment) |
| Back-pressure | Manual | Built-in (Flush threshold) |
| Allocations | Often per-read | Pooled, near-zero |
| Complexity | Simpler | Steeper learning curve |
| Best for | General I/O, files | High-perf network protocol parsing |

Use **Streams** for files and ordinary I/O. Use **Pipelines** when you're writing a server that parses a wire protocol (HTTP, custom TCP, etc.) at scale.

---

## Common bugs

### Wrong AdvanceTo arguments

```csharp
// ⚠ — claiming you consumed everything when you only examined it
reader.AdvanceTo(buffer.End);   // pipe discards unparsed partial message → data loss

// ✓ — consumed only complete messages, examined all
reader.AdvanceTo(consumedPosition, buffer.End);
```

Getting consumed/examined wrong causes either data loss (consumed too much) or infinite spinning (examined too little, ReadAsync returns immediately with the same data).

### Forgetting to Complete

```csharp
await reader.CompleteAsync();   // signal done — otherwise the writer side hangs
await writer.CompleteAsync();
```

Both ends must `Complete` (often in `finally` / passing exceptions) or the counterpart waits forever.

### Treating ReadOnlySequence as contiguous

```csharp
var span = buffer.FirstSpan;   // ⚠ — only the FIRST segment; misses the rest if multi-segment
```

Check `IsSingleSegment` or use `SequenceReader`.

---

## Performance notes

- Buffers come from a pool (`MemoryPool<byte>`) — no per-read allocation.
- No copying of leftover bytes between reads (segments are chained).
- Back-pressure bounds memory automatically.
- `SequenceReader<byte>` parses across segments without materializing a contiguous buffer.
- Kestrel uses pipelines to handle millions of requests/sec with minimal GC.

---

## When to use Pipelines

- Writing a network server / protocol parser (TCP, HTTP, gRPC).
- Streaming parse of large data where allocation matters.
- Producer/consumer with back-pressure needs.

When **not** to:
- Reading a config file (use `File.ReadAllText`).
- Simple stream copy (use `Stream.CopyToAsync`).
- You don't have a measured perf problem — pipelines add complexity.

---

## Summary

- `System.IO.Pipelines` is a high-performance, low-allocation I/O library for streaming parse.
- `PipeWriter` produces into pooled buffers; `PipeReader` consumes a `ReadOnlySequence<byte>`.
- `AdvanceTo(consumed, examined)` handles partial messages without copying.
- Back-pressure is built in via flush thresholds.
- Powers Kestrel, SignalR, gRPC. Use for high-scale protocol parsing — not for ordinary file I/O.

→ Next: [04-TextReadersWriters.md](04-TextReadersWriters.md)
