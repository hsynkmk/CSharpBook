# ArrayPool&lt;T&gt;

## What it is

`ArrayPool<T>.Shared` is a **thread-safe pool of reusable arrays**. Instead of `new byte[size]` (which allocates), you `Rent(size)` from the pool. When done, `Return` puts it back. Avoids heap pressure for buffer-heavy code.

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
try {
    int read = stream.Read(buffer, 0, 1024);
    Process(buffer, read);
} finally {
    ArrayPool<byte>.Shared.Return(buffer);
}
```

Critical for high-throughput servers (Kestrel, gRPC, SignalR all use it). For periodic buffer needs, ArrayPool is dramatically more efficient than per-call allocation.

---

## Why it exists

Allocating arrays in a hot loop:

```csharp
for (int i = 0; i < 100_000; i++) {
    byte[] buf = new byte[1024];
    ReadFrom(stream, buf);
    Process(buf);
}
// 100K allocations × 1 KB = 100 MB of garbage; many Gen0 GCs
```

For arrays ≥ 85,000 bytes, they go to the LOH — even worse (not compacted, only collected in Gen2).

ArrayPool fixes both:

```csharp
for (int i = 0; i < 100_000; i++) {
    byte[] buf = ArrayPool<byte>.Shared.Rent(1024);
    try {
        ReadFrom(stream, buf);
        Process(buf);
    } finally {
        ArrayPool<byte>.Shared.Return(buf);
    }
}
// Reuses the same handful of arrays. Near-zero GC pressure.
```

---

## API

```csharp
// Rent
byte[] buf = ArrayPool<byte>.Shared.Rent(1024);   // returns array of length >= 1024
// buf might be 1024, 1500, 2048 — depends on the pool's internal buckets

// Return
ArrayPool<byte>.Shared.Return(buf);                  // back to the pool
ArrayPool<byte>.Shared.Return(buf, clearArray: true); // zero before returning (for sensitive data)
```

`Shared` is a singleton pool. For custom configuration, create your own:

```csharp
var customPool = ArrayPool<byte>.Create(maxArrayLength: 1024 * 1024, maxArraysPerBucket: 50);
```

The custom pool is **independent** of `Shared` — separate memory budget.

---

## Critical rule: `Rent(n)` may return a larger array

`Rent(1000)` might return an array of length 1024 (the next bucket size). You're not guaranteed exactly the requested length.

```csharp
byte[] buf = ArrayPool<byte>.Shared.Rent(1000);
Console.WriteLine(buf.Length);   // could be 1024 or more
```

**Implication**: use the **requested size**, not `buf.Length`, for reading and processing:

```csharp
const int wanted = 1000;
byte[] buf = ArrayPool<byte>.Shared.Rent(wanted);
try {
    int read = stream.Read(buf, 0, wanted);   // not buf.Length!
    string text = Encoding.UTF8.GetString(buf, 0, read);
} finally {
    ArrayPool<byte>.Shared.Return(buf);
}
```

Iterating `for (int i = 0; i < buf.Length; ...)` walks past your data into uninitialized (or previous-renter's) memory. Always track and use the logical size separately.

For Span:
```csharp
Span<byte> span = buf.AsSpan(0, wanted);   // explicit, the logical size
```

---

## Rent/Return must be paired — and use try/finally

```csharp
byte[] buf = ArrayPool<byte>.Shared.Rent(1024);
try {
    // ... use buf ...
} finally {
    ArrayPool<byte>.Shared.Return(buf);
}
```

If you don't Return, the array is gone (still alive as long as you hold the reference; never reused). It becomes a leak in the pool's perspective — the pool eventually allocates fresh arrays to replace it.

If you Return twice, the pool may give the same array to two renters → corruption.

**Always use try/finally** (or a custom `IDisposable` wrapper).

---

## `clearArray: true` — for sensitive data

If the array held secrets (passwords, keys, PII), zero it before returning so the next renter doesn't see your data:

```csharp
ArrayPool<byte>.Shared.Return(buf, clearArray: true);
```

The pool zeros the array. Slightly slower than skipping.

For non-sensitive data, don't bother — leaving stale bytes is fine since the renter overwrites them anyway.

`Rent` doesn't zero-init either — you receive whatever the previous renter left. This is part of why it's fast.

---

## Custom IDisposable wrapper

Pair Rent + Return automatically:

```csharp
public sealed class PooledArray<T> : IDisposable {
    private T[]? _array;
    public Span<T> Span { get; }

    public PooledArray(int size) {
        _array = ArrayPool<T>.Shared.Rent(size);
        Span = _array.AsSpan(0, size);
    }

    public void Dispose() {
        if (_array is not null) {
            ArrayPool<T>.Shared.Return(_array);
            _array = null;
        }
    }
}

// Use
using var p = new PooledArray<byte>(1024);
ReadInto(p.Span);
Process(p.Span);
// auto-Return at end of using
```

Many libraries (e.g., the JSON serializer internals) use similar wrappers.

---

## When to use ArrayPool

✓ Buffers used **transiently** (rent, use, return — quick lifetime).
✓ Large arrays (especially ≥ 85,000 bytes — avoids LOH).
✓ High-frequency code: parsing, network I/O, formatting.
✓ Anywhere you'd otherwise see allocation churn in profiler.

✗ Long-held buffers — the pool can't reclaim them.
✗ Sizes that fit in `stackalloc` (small temporary) — `stackalloc` is faster.
✗ When you actually need an array you keep (not borrowing).

---

## ArrayPool vs MemoryPool

```csharp
// ArrayPool — for arrays
byte[] buf = ArrayPool<byte>.Shared.Rent(1024);
try { /* ... */ } finally { ArrayPool<byte>.Shared.Return(buf); }

// MemoryPool — for Memory<T>
using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(1024);
Memory<byte> mem = owner.Memory;
```

`MemoryPool<T>` returns an `IMemoryOwner<T>` (dispose to return). Useful when the consumer wants `Memory<T>` (async-friendly). Internally, MemoryPool may use ArrayPool.

For most cases: use ArrayPool + `.AsMemory()` / `.AsSpan()`. Simpler.

---

## Internals — how the Shared pool works

`ArrayPool<T>.Shared` is implemented as **per-CPU buckets** (since .NET Core 2.1). Each bucket holds arrays of a specific size class:

| Bucket | Size class |
|---|---|
| 0 | 16 |
| 1 | 32 |
| 2 | 64 |
| ... | (doubles each) |
| 16 | ~1 MB |

A `Rent(1000)` rounds up to bucket 7 (1024). Returns an array from that bucket if available; else allocates a new one.

**Per-CPU** locality reduces contention. Each thread mostly rents from its CPU's bucket. Returns also stay local.

For very-large requests (above the largest bucket), the pool falls back to plain `new T[size]` and doesn't store it back. So `Rent(10_000_000)` allocates — pool size limits matter.

### Memory bound

The pool has a max-arrays-per-bucket limit. Once exceeded, additional Returns are dropped (the returned array is left for GC). Prevents the pool from growing unboundedly.

This means: if you Rent many more than Return (i.e., leak), the pool just allocates fresh ones.

### Per-CPU vs cross-thread

If thread A rents and thread B returns:
- B might return to its own CPU's bucket, not A's.
- Mostly fine — eventually balances.

For very-bursty asymmetric patterns, custom pools can be tuned.

---

## .NET 8+ improvements

- **`ArrayPool<T>.Shared`** in .NET 8 is more aggressive about per-CPU caching.
- **Smaller per-size buckets** by default.
- **`ConfigurableArrayPool`** (third-party / library) for ultimate control.

For typical use, the default `Shared` pool is excellent.

---

## Common patterns

### Read from stream

```csharp
public async Task<byte[]> ReadAllAsync(Stream stream, CancellationToken ct = default) {
    var buf = ArrayPool<byte>.Shared.Rent(8192);
    try {
        using var ms = new MemoryStream();
        int read;
        while ((read = await stream.ReadAsync(buf.AsMemory(0, 8192), ct)) > 0) {
            ms.Write(buf, 0, read);
        }
        return ms.ToArray();
    } finally {
        ArrayPool<byte>.Shared.Return(buf);
    }
}
```

Read in 8 KB chunks using a rented buffer. The MemoryStream collects the result; the rented buffer is reused per call.

### Formatting large output

```csharp
public string FormatBig(int[] data) {
    var buf = ArrayPool<char>.Shared.Rent(data.Length * 12);
    try {
        int written = 0;
        foreach (var n in data) {
            n.TryFormat(buf.AsSpan(written), out int w);
            written += w;
            buf[written++] = ',';
        }
        return new string(buf.AsSpan(0, written - 1));   // exclude trailing comma
    } finally {
        ArrayPool<char>.Shared.Return(buf);
    }
}
```

Rent a char[], format into it, materialize the final string at the end. One allocation (the string), vs many with naive StringBuilder/string concat.

### Custom Pooled struct

```csharp
public readonly ref struct PooledArrayHolder<T> {
    private readonly T[] _array;
    public Span<T> Span { get; }

    public PooledArrayHolder(int size) {
        _array = ArrayPool<T>.Shared.Rent(size);
        Span = _array.AsSpan(0, size);
    }

    public void Dispose() => ArrayPool<T>.Shared.Return(_array);
}

// Use
public void Process() {
    using var p = new PooledArrayHolder<byte>(1024);   // ref struct using
    // ...
}
```

`ref struct using` is supported (C# 8+ "using disposable pattern" — type doesn't need to implement IDisposable; the compiler calls `Dispose` if the type has a void `Dispose()` method).

---

## Common bugs

### Using `buf.Length` instead of requested size

```csharp
byte[] buf = ArrayPool<byte>.Shared.Rent(1000);
for (int i = 0; i < buf.Length; i++) {   // ⚠ — buf.Length might be 1024
    Process(buf[i]);
}
```

Track the logical size separately:
```csharp
const int wanted = 1000;
byte[] buf = ArrayPool<byte>.Shared.Rent(wanted);
for (int i = 0; i < wanted; i++) Process(buf[i]);
```

### Forgetting to Return

```csharp
byte[] buf = ArrayPool<byte>.Shared.Rent(1024);
DoSomething(buf);
// no Return — array is "leaked" from pool perspective (will be replaced)
```

Always use try/finally. Or a wrapper IDisposable.

### Returning twice

```csharp
ArrayPool<byte>.Shared.Return(buf);
// ... continue using buf ...
ArrayPool<byte>.Shared.Return(buf);   // ⚠ pool may give buf to someone else
```

The second Return can corrupt the pool. Use a wrapper that nulls the reference after Return.

### Holding the array after Return

```csharp
ArrayPool<byte>.Shared.Return(buf);
Process(buf);   // ⚠ — another renter may have it now
```

After Return, treat the array as if it were freed.

### Sensitive data not cleared

```csharp
var buf = ArrayPool<byte>.Shared.Rent(1024);
ReadPasswordInto(buf);
ArrayPool<byte>.Shared.Return(buf);   // ⚠ next renter sees the password bytes
```

Use `Return(buf, clearArray: true)` for secrets.

---

## Performance

- `Rent`: ~50-100 ns (lookup in per-CPU bucket).
- `Return`: ~50-100 ns.
- vs `new byte[1024]`: ~10 ns, but no GC pressure with ArrayPool. Net win when allocation rate is high.
- For 100K iterations of "rent + use + return": dramatically faster than 100K allocations + collections.

The win isn't per-operation cost; it's **eliminating GC churn**. Profile: look at Gen0 collection count before vs after.

---

## When ArrayPool isn't worth it

- For tiny allocations (16 bytes) — Gen0 handles them efficiently.
- For long-lived buffers (you allocate once, keep forever) — no churn to avoid.
- For low-frequency allocations — the complexity isn't worth a measurable win.

For high-throughput servers, parsers, formatters, network code — ArrayPool is essential.

---

## Summary

- `ArrayPool<T>.Shared` is a thread-safe pool of reusable arrays.
- Rent + Return (with try/finally) to borrow and return.
- `Rent(n)` may return larger — use the requested size for processing.
- Pair with `Span<T>.Slice` for the logical view.
- Avoids LOH thrashing for large arrays.
- For sensitive data, use `Return(buf, clearArray: true)`.
- Used pervasively in modern BCL — Kestrel, JSON, gRPC, SignalR.

→ Next: [08-Stackalloc.md](08-Stackalloc.md)
