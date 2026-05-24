# IAsyncDisposable and `await using`

## What it is

`IAsyncDisposable` is the async version of `IDisposable` — for types whose cleanup itself involves async work (network flush, database connection close, async stream dispose).

```csharp
public interface IAsyncDisposable {
    ValueTask DisposeAsync();
}
```

Consumed via `await using`:

```csharp
await using var conn = await OpenConnectionAsync();
// ... use conn ...
// At end of scope: await conn.DisposeAsync()
```

Added in C# 8 (2019). Used widely in modern I/O — `Stream`, `HttpResponseMessage`, `DbContext`, `IAsyncEnumerator<T>`, gRPC clients, Channel readers.

---

## Why it exists

`IDisposable.Dispose` is synchronous. For resources whose cleanup must do real work (flush pending writes, close a TLS connection cleanly, await a graceful shutdown), forcing it into sync mode meant:
- Blocking a thread for the cleanup.
- Or doing the work sloppily / fire-and-forget.

`IAsyncDisposable` exposes the inherent async nature of cleanup:

```csharp
public ValueTask DisposeAsync() {
    return _stream.FlushAsync();   // properly await the flush
}
```

The caller awaits the dispose. Cleanup completes cleanly before resources are released or scope exits.

---

## Basics

### Implementing

```csharp
public class MyResource : IAsyncDisposable {
    private readonly Stream _stream;
    private bool _disposed;

    public MyResource(Stream stream) { _stream = stream; }

    public async ValueTask DisposeAsync() {
        if (_disposed) return;
        _disposed = true;
        await _stream.DisposeAsync();
        // any other async cleanup
    }
}
```

`DisposeAsync` returns `ValueTask` (not `Task` — for the sync-fast-path optimization).

### Consuming with `await using`

```csharp
await using (var r = new MyResource(stream)) {
    await r.DoSomethingAsync();
}
// r.DisposeAsync() awaited here

// Or declaration form:
await using var r = new MyResource(stream);
await r.DoSomethingAsync();
// r.DisposeAsync() awaited at end of enclosing scope
```

Same shape as `using` and `using var`, with `await` prefix.

### Mixing IDisposable and IAsyncDisposable

A type can implement both:

```csharp
public class Resource : IDisposable, IAsyncDisposable {
    public void Dispose() { /* sync cleanup */ }
    public async ValueTask DisposeAsync() { /* async cleanup */ }
}
```

For consumers:
- `using` → calls `Dispose()`.
- `await using` → calls `DisposeAsync()`.

The compiler picks based on which form you wrote.

For types that have only one form, the consumer uses the matching keyword.

---

## When to use which

```csharp
// Resource implements only IDisposable
using var stream = new FileStream(...);

// Resource implements only IAsyncDisposable
await using var conn = await OpenAsync();

// Resource implements both
await using var ctx = new DbContext();   // prefer async
// or
using var ctx = new DbContext();         // sync, blocks the thread
```

When both exist, prefer `await using` in async code — it's the proper async cleanup. `using` is fine if you're in sync code and the type's sync Dispose works correctly.

---

## DbContext, HttpClient, Stream — the BCL adoption

EF Core 2.1+:
```csharp
await using var ctx = new AppDbContext();
var users = await ctx.Users.ToListAsync();
// ctx.DisposeAsync() awaited
```

HttpClient:
```csharp
await using var resp = await httpClient.GetAsync(url);
// resp.DisposeAsync() awaited
```

Streams:
```csharp
await using var fs = File.OpenRead(path);
// fs.DisposeAsync() awaited — properly flushes buffered writes if any
```

Channel readers and other async-heavy primitives also implement IAsyncDisposable.

When in doubt, prefer `await using` over `using` for I/O-bound resources.

---

## Full dispose pattern with both interfaces

For types managing both managed (async-cleanup) and unmanaged resources:

```csharp
public class BigResource : IDisposable, IAsyncDisposable {
    private FileStream? _stream;
    private IntPtr _native;
    private bool _disposed;

    public void Dispose() {
        if (_disposed) return;
        _stream?.Dispose();
        FreeNative();
        _disposed = true;
        GC.SuppressFinalize(this);
    }

    public async ValueTask DisposeAsync() {
        if (_disposed) return;
        if (_stream is not null) {
            await _stream.DisposeAsync();
            _stream = null;
        }
        FreeNative();
        _disposed = true;
        GC.SuppressFinalize(this);
    }

    ~BigResource() => Dispose();   // unmanaged cleanup fallback

    private void FreeNative() {
        if (_native != IntPtr.Zero) {
            Marshal.FreeHGlobal(_native);
            _native = IntPtr.Zero;
        }
    }
}
```

Both `Dispose` and `DisposeAsync` end up doing the same fundamental work. The async path uses async-friendly inner cleanup.

The finalizer is a safety net for unmanaged resources if neither dispose method runs (forgotten by caller).

---

## ValueTask for DisposeAsync

`DisposeAsync` returns `ValueTask` (not `Task`) — saves an allocation when cleanup is synchronous.

```csharp
public ValueTask DisposeAsync() {
    if (_disposed) return ValueTask.CompletedTask;   // synchronous fast path
    // ... async cleanup ...
    return new ValueTask(_stream.DisposeAsync());
}
```

`ValueTask.CompletedTask` is a static no-op — zero allocation.

For most cases:
```csharp
public async ValueTask DisposeAsync() {
    await _stream.DisposeAsync();   // compiler handles ValueTask wrapping
}
```

Compiler generates the right state machine.

---

## Async iteration disposal

`IAsyncEnumerator<T>` implements `IAsyncDisposable`. `await foreach` calls `DisposeAsync` at the end:

```csharp
await foreach (var x in source) { /* ... */ }
// Implicit: await source's enumerator DisposeAsync()
```

This is why iterator methods can have `try/finally` cleanup:

```csharp
public async IAsyncEnumerable<int> Stream() {
    using var resource = AcquireExpensive();
    for (int i = 0; i < 100; i++) {
        await Task.Delay(100);
        yield return i;
    }
    // resource.Dispose() runs when the enumerator is disposed
}
```

When the consumer's `await foreach` exits (normally or via exception), the enumerator's DisposeAsync runs the `finally`, disposing `resource`.

---

## Exception handling in DisposeAsync

`DisposeAsync` SHOULD NOT throw (best practice). But it CAN. If it does during `await using`:

```csharp
await using (var x = new X()) {
    throw new Exception("oops");   // throws here
}
// DisposeAsync is still called
// If DisposeAsync ALSO throws, the original exception is what bubbles up
// (the dispose exception is suppressed unless the original was clean)
```

The behavior mirrors `using` blocks. The original exception wins; the dispose exception is suppressed (for normal completion paths, the dispose exception bubbles up).

**Convention**: never throw from DisposeAsync if you can avoid it. Cleanup failures should be logged, not thrown.

---

## Internals — what the compiler generates

```csharp
await using var x = new X();
// ... use x ...
```

Roughly compiles to:

```csharp
var x = new X();
try {
    // ... use x ...
} finally {
    if (x is not null) await x.DisposeAsync();
}
```

Same try/finally pattern as `using`, but with `await` on the disposal. The state machine for the enclosing method handles the resumption after the dispose completes.

If x is null, DisposeAsync isn't called (avoids NRE).

---

## Common patterns

### Service that owns a connection

```csharp
public class MyService : IAsyncDisposable {
    private readonly DbConnection _conn;
    public MyService(DbConnection conn) { _conn = conn; }
    public async ValueTask DisposeAsync() => await _conn.DisposeAsync();
}

await using var svc = new MyService(...);
```

### Lazy resource initialization

```csharp
public class LazyResource : IAsyncDisposable {
    private Stream? _stream;
    private bool _disposed;

    public async ValueTask<Stream> GetStreamAsync(CancellationToken ct = default) {
        if (_disposed) throw new ObjectDisposedException(GetType().Name);
        _stream ??= await OpenStreamAsync(ct);
        return _stream;
    }

    public async ValueTask DisposeAsync() {
        _disposed = true;
        if (_stream is not null) await _stream.DisposeAsync();
    }
}
```

### Composite

```csharp
public class Composite : IAsyncDisposable {
    private readonly List<IAsyncDisposable> _resources = new();
    public void Track(IAsyncDisposable r) => _resources.Add(r);
    public async ValueTask DisposeAsync() {
        foreach (var r in _resources) {
            try { await r.DisposeAsync(); }
            catch { /* log but continue */ }
        }
    }
}
```

Dispose-all pattern. Errors logged but not propagated to avoid losing all cleanup on the first failure.

---

## Common bugs

### Forgetting to await DisposeAsync

```csharp
var r = new Resource();
r.DisposeAsync();   // ⚠ — returned ValueTask discarded
```

Without await, the disposal might not complete before the variable is GC'd or before the next operation. Use `await using` or `await r.DisposeAsync()` explicitly.

### Mixing `using` and `await using` inconsistently

```csharp
public async Task M() {
    using var stream = File.OpenRead(...);   // sync dispose
    var data = await stream.ReadAsync(...);
}   // stream.Dispose() — but FileStream implements DisposeAsync that's better
```

For `Stream`, `DbContext`, and other async-aware types, prefer `await using`.

### Calling Dispose() but type only implements IAsyncDisposable

```csharp
var x = new AsyncOnlyResource();
x.Dispose();   // ⚠ compile error — no Dispose method
```

Use `await using` or `await x.DisposeAsync()`.

### DisposeAsync that captures state with locks

```csharp
public async ValueTask DisposeAsync() {
    lock (_lock) {   // ⚠ — can't have `lock` in async method
        // ...
    }
}
```

`lock` blocks can't cross `await`. Use `SemaphoreSlim` or `Monitor.Enter/Exit` (carefully — see [§09 LockingPrimitives](09-LockingPrimitives.md)).

### Long-running async work in DisposeAsync

```csharp
public async ValueTask DisposeAsync() {
    await DoLengthyAsync();   // ⚠ — cleanup shouldn't take seconds
}
```

DisposeAsync should be fast. Long async cleanup blocks the `await using` exit. Consider a separate `StopAsync` method for long-running shutdown, with DisposeAsync as a final guard.

---

## When to use IAsyncDisposable

✓ Type owns I/O resources (network, database, async streams).
✓ Cleanup involves awaiting (flush, drain, graceful shutdown).
✓ Type implements `IAsyncEnumerable<T>` (you'll need DisposeAsync for the enumerator anyway).

✗ Type holds only managed memory or trivial sync resources — IDisposable suffices.
✗ Cleanup is purely CPU work — no async needed.

---

## Summary

- `IAsyncDisposable` for types whose cleanup needs `await`.
- `await using` to consume them.
- Implement both `IDisposable` and `IAsyncDisposable` for max flexibility on types that can be used sync or async.
- DisposeAsync returns `ValueTask` (not Task).
- Should not throw.
- The BCL widely uses it now — Streams, HttpResponseMessage, DbContext, Channels.

→ Next: [09-LockingPrimitives.md](09-LockingPrimitives.md)
