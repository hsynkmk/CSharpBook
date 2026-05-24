# Memory&lt;T&gt; and ReadOnlyMemory&lt;T&gt;

## What it is

`Memory<T>` is `Span<T>`'s heap-friendly cousin. Same conceptual API (slice, index, length), but **not a ref struct** — you can store it in fields, capture it in lambdas, pass it across `await`. Trade-off: it's heavier (16 bytes + the underlying storage info) and slightly slower to access.

```csharp
private Memory<byte> _buffer;   // ✓ — Memory as a field is allowed

public async Task ProcessAsync() {
    Memory<byte> mem = _buffer;
    await SomeAsync();          // ✓ — Memory can cross await
    Span<byte> span = mem.Span;  // get Span when you need to use it
}
```

Used for storing buffers that live across async boundaries. Channel<T>, System.IO.Pipelines, gRPC, Kestrel all use `Memory<T>` for async-friendly buffer references.

---

## Why both Span and Memory exist

`Span<T>` is fast but restricted (stack-only). `Memory<T>` is slightly slower but unrestricted.

Use Memory when:
- You need to **store** a buffer in a field or collection.
- The buffer crosses an `await`.
- You're passing through a lambda or `IEnumerable<T>`.

Use Span otherwise — faster on hot paths.

The standard pattern: **store as Memory, use as Span** for the actual work.

```csharp
public async Task<int> ProcessAsync(Memory<byte> buffer, CancellationToken ct = default) {
    await SomeIoAsync(ct);

    Span<byte> span = buffer.Span;   // get the Span for fast access
    int sum = 0;
    foreach (var b in span) sum += b;
    return sum;
}
```

---

## Creating Memory

```csharp
// From array
byte[] arr = new byte[1024];
Memory<byte> mem = arr;
Memory<byte> slice = arr.AsMemory(100, 500);   // index, length

// From string (ReadOnlyMemory<char>)
ReadOnlyMemory<char> s = "Hello".AsMemory();

// From IMemoryOwner<T> (rented from a pool)
using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(1024);
Memory<byte> mem = owner.Memory;

// From an unmanaged pointer (advanced)
unsafe {
    void* ptr = NativeMethods.Allocate(1024);
    var mem = new UnmanagedMemoryStream((byte*)ptr, 1024);
    // ... MemoryManager<T> for safe wrappers
}
```

---

## Memory.Span — the bridge

When you have Memory and need fast access, get the Span:

```csharp
Memory<byte> mem = buffer;
Span<byte> span = mem.Span;   // O(1) — just unpacks the storage info

// Now use Span operations
span.Fill(0);
span[0] = 42;
```

`.Span` is cheap — no allocation, just unpacks the memory descriptor. Get the Span at the point of use, do your fast work, drop the Span when done.

Don't store a Span after getting it from Memory — that defeats the point. Get it per operation.

---

## Slicing

Same as Span:

```csharp
Memory<byte> mem = buffer.AsMemory();
Memory<byte> first = mem[..100];
Memory<byte> rest = mem[100..];
Memory<byte> middle = mem[1..^1];
Memory<byte> slice = mem.Slice(100, 50);
```

All O(1), no copy. Same semantics as Span slicing.

---

## ReadOnlyMemory&lt;T&gt;

The read-only variant.

```csharp
ReadOnlyMemory<char> s = "Hello".AsMemory();
ReadOnlySpan<char> span = s.Span;
```

`string` implicitly converts to `ReadOnlyMemory<char>` — used widely in async parsing scenarios.

---

## Storing Memory in a field

```csharp
public class StreamingProcessor {
    private Memory<byte> _buffer;
    public StreamingProcessor() {
        _buffer = new byte[8192];
    }

    public async Task ReadAsync(Stream stream, CancellationToken ct) {
        int read = await stream.ReadAsync(_buffer, ct);   // Span would NOT work here
        ProcessSpan(_buffer.Span[..read]);
    }

    private void ProcessSpan(Span<byte> data) {
        // fast sync work
    }
}
```

The buffer is a class field. Span couldn't be — would compile error. Memory works.

`Stream.ReadAsync(Memory<byte>)` is the modern overload — accepts Memory, lets you pass async-friendly buffers.

---

## Passing Memory across await

```csharp
public async Task<int> CountBytesAsync(Memory<byte> data) {
    await Task.Delay(100);          // ✓ — Memory survives the await
    return data.Span.Length;
}
```

Memory isn't a `ref struct`, so the async state machine can capture it. The runtime tracks the underlying storage; when the continuation runs, the Memory is still valid.

For Span, the equivalent would be a compile error.

---

## MemoryPool&lt;T&gt; — pooled memory

For pooled buffers in async code:

```csharp
using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(1024);
Memory<byte> buffer = owner.Memory;

await stream.ReadAsync(buffer);
ProcessSpan(buffer.Span);
```

`IMemoryOwner<T>` wraps a rented Memory with `IDisposable` semantics. When you Dispose, the buffer is returned to the pool.

Differs from `ArrayPool<T>.Shared.Rent` (next file) — `MemoryPool` is designed for the Memory abstraction; `ArrayPool` is for raw arrays.

For most code, `ArrayPool` + `.AsMemory()` is just as good and slightly simpler.

---

## Memory in System.IO.Pipelines

```csharp
PipeReader reader = ...;
while (true) {
    ReadResult result = await reader.ReadAsync();
    ReadOnlySequence<byte> buffer = result.Buffer;
    // ... process buffer ...
    reader.AdvanceTo(buffer.End);
    if (result.IsCompleted) break;
}
```

`ReadOnlySequence<byte>` is a collection of `ReadOnlyMemory<byte>` segments — used for zero-copy I/O processing. Internal use of Memory.

---

## Internals — what Memory holds

`Memory<T>` is a struct with three fields:

```csharp
public readonly struct Memory<T> {
    private readonly object? _object;   // could be T[], string, or MemoryManager<T>
    private readonly int _index;
    private readonly int _length;
}
```

- `_object` is what the Memory references. For arrays, it's the T[]. For strings (in `ReadOnlyMemory<char>`), it's the string. For pooled / unmanaged, it's a `MemoryManager<T>`.
- `_index` is the offset within `_object`.
- `_length` is the slice length.

When you ask for `.Span`, the runtime constructs a Span by looking at `_object`'s type and computing the right pointer:
- For arrays: pointer to the array's element at `_index`.
- For string: pointer to the char at `_index`.
- For MemoryManager: calls `GetSpan()`.

The dispatch adds a few ns vs raw Span — that's why Span is preferred for tight loops.

### MemoryManager&lt;T&gt;

For custom underlying memory (unmanaged, GPU buffer, etc.), inherit from `MemoryManager<T>`:

```csharp
public class NativeMemoryManager : MemoryManager<byte> {
    private unsafe byte* _ptr;
    private int _length;

    public unsafe NativeMemoryManager(int length) {
        _length = length;
        _ptr = (byte*)NativeMemory.Alloc((nuint)length);
    }

    public override unsafe Span<byte> GetSpan() => new Span<byte>(_ptr, _length);
    public override MemoryHandle Pin(int elementIndex = 0) {
        return new MemoryHandle(_ptr + elementIndex);
    }
    public override void Unpin() { /* no-op for our case */ }
    protected override unsafe void Dispose(bool disposing) {
        NativeMemory.Free(_ptr);
        _ptr = null;
    }
}
```

Used to wrap native buffers, GPU memory, memory-mapped files. Then your code uses `Memory<byte>` uniformly without caring about the backing storage.

---

## Common patterns

### Async I/O with pooled buffer

```csharp
public async Task<byte[]> ReadStreamAsync(Stream stream, CancellationToken ct = default) {
    using var owner = MemoryPool<byte>.Shared.Rent(8192);
    var buffer = owner.Memory;

    using var ms = new MemoryStream();
    int read;
    while ((read = await stream.ReadAsync(buffer, ct)) > 0) {
        ms.Write(buffer.Span[..read]);
    }
    return ms.ToArray();
}
```

Rented buffer, async read, slice the filled portion, copy to output. The pool reuses the buffer.

### Storing in a Channel<T>

```csharp
var channel = Channel.CreateBounded<ReadOnlyMemory<byte>>(100);

// Producer
await channel.Writer.WriteAsync(buffer.AsMemory(0, read));

// Consumer
await foreach (var mem in channel.Reader.ReadAllAsync()) {
    Process(mem.Span);
}
```

You can store `Memory<T>` in collections, channels, queues. Can't with Span.

### Returning Memory from a method

```csharp
public Memory<byte> GetHeader() {
    return _data.AsMemory(0, 32);
}

// Caller
Memory<byte> header = svc.GetHeader();
// ... can store, await, manipulate ...
ReadOnlySpan<byte> span = header.Span;
Console.WriteLine(span.Length);
```

Method returns a Memory pointing into the service's buffer. Caller can do whatever. As long as `_data` isn't reclaimed/reused while the caller holds it, safe.

---

## ReadOnlySequence&lt;T&gt;

For data spread across **multiple** Memory segments (common in network/pipeline scenarios):

```csharp
ReadOnlySequence<byte> sequence = reader.Buffer;
foreach (ReadOnlyMemory<byte> segment in sequence) {
    ProcessSegment(segment.Span);
}
```

`ReadOnlySequence<byte>` is what `PipeReader` produces — a linked list of memory blocks. Lets you process data as it streams in without copying.

`System.Buffers` namespace has helpers (`SequenceReader<T>`) for parsing across segments.

---

## Common bugs

### Storing a Span instead of Memory

```csharp
public class Cache {
    private Span<byte> _data;   // ❌ compile error
}
```

Use Memory:
```csharp
private Memory<byte> _data;   // ✓
```

### Forgetting to dispose IMemoryOwner

```csharp
IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(1024);
// ... use owner.Memory ...
// missing owner.Dispose() — buffer leaks back to the pool
```

Use `using`.

### Mutating Memory while async is using it

```csharp
byte[] arr = new byte[1024];
Memory<byte> mem = arr;
_ = stream.ReadAsync(mem);   // async read in progress
arr = new byte[1024];        // didn't help — mem still points to the OLD array
```

Memory captures the underlying array reference. Reassigning the variable doesn't free anything.

But this CAN cause problems:
```csharp
Memory<byte> mem = list.ToArray().AsMemory();
// list disposed; new list filled
// mem still points to the OLD array — safe (GC'd if no other refs)
```

For long-lived Memory backed by pooled buffers, ensure no one returns the buffer to the pool while you still have references.

---

## Span vs Memory comparison

|  | Span&lt;T&gt; | Memory&lt;T&gt; |
|---|---|---|
| Type | `ref struct` | regular struct |
| Class field allowed | ✗ | ✓ |
| Cross `await` | ✗ | ✓ |
| Captured by lambda | ✗ (unless static) | ✓ |
| Boxed / object | ✗ | ✓ |
| Access cost | array-fast | slightly slower (indirection) |
| Slice cost | O(1) | O(1) |
| Restrictions | many | none |

**Idiom**: store Memory, use Span. Take `Memory<T>` as a parameter; convert to `Span<T>` inside the method for hot work.

---

## When to use Memory

✓ Async I/O buffers.
✓ Pooled storage you'll use across await.
✓ Returning a "view" from a method into a long-lived buffer.
✓ Channel/Queue elements.
✓ Class fields holding buffer references.

✗ Pure synchronous hot loops — Span is faster.
✗ Stack-allocated buffers — those must be Span.
✗ Quick local slicing — Span is enough.

---

## Performance

- Memory creation: ~1 ns (struct construction).
- `.Span` accessor: ~few ns (dispatch on _object's type).
- Slicing: O(1), few ns.
- For tight loops, do `var span = mem.Span;` once outside the loop, iterate the span.

For most async I/O code, the per-access overhead of Memory is dwarfed by the I/O cost. No concern.

---

## Summary

- `Memory<T>` is a heap-friendly `Span<T>` — same conceptual view, no `ref struct` restrictions.
- Use Memory for async buffers, fields, collections.
- Use Span for hot work — get from Memory via `.Span`.
- The pattern: store as Memory, work as Span.
- `ReadOnlyMemory<char>` is the immutable variant; `string` converts implicitly.
- `IMemoryOwner<T>` + `MemoryPool<T>` for pooled async buffers.

→ Next: [07-ArrayPool.md](07-ArrayPool.md)
