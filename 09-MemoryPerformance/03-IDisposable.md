# IDisposable and the Dispose Pattern

## What it is

`IDisposable` is the contract for **deterministic resource cleanup**. Implementing types release resources (file handles, sockets, locks, unmanaged memory) when their `Dispose()` method is called — usually via the `using` statement.

```csharp
public interface IDisposable {
    void Dispose();
}

using (var stream = File.OpenRead("data.txt")) {
    // ... use stream ...
}
// stream.Dispose() called here — file handle freed
```

GC handles managed memory automatically. `IDisposable` handles **non-memory resources** the GC doesn't know about: OS handles, native memory, locks, network connections. These need timely cleanup that can't wait for the next GC.

---

## Why it exists

```csharp
var stream = File.OpenRead("data.txt");
// ... use stream ...
// Forget to close?
```

The OS file handle remains open. Eventually, the GC will run the finalizer (which closes it), but:
- That might take minutes.
- The handle blocks other processes from accessing the file.
- For high-concurrency code, you'd exhaust the per-process handle limit.

`IDisposable` fixes this: you (or the `using` block) explicitly release the resource the moment it's no longer needed. Doesn't wait for GC.

---

## Basic usage with `using`

```csharp
// Block form
using (var stream = File.OpenRead("data.txt")) {
    var bytes = new byte[100];
    stream.Read(bytes, 0, 100);
}   // stream.Dispose() called here

// Declaration form (C# 8+) — implicit scope = enclosing block
public void M() {
    using var stream = File.OpenRead("data.txt");
    // ... stream usable through rest of method ...
}   // stream.Dispose() called here
```

Both translate to:

```csharp
var stream = File.OpenRead("data.txt");
try {
    // ... use stream ...
} finally {
    stream?.Dispose();
}
```

`using` ensures Dispose is called even if the body throws.

---

## Common IDisposable types

- **Streams**: `FileStream`, `MemoryStream`, `BufferedStream`, etc.
- **Readers/Writers**: `StreamReader`, `StreamWriter`, `BinaryReader`, ...
- **HTTP**: `HttpClient`, `HttpResponseMessage`, `HttpRequestMessage`.
- **Database**: `DbConnection`, `DbCommand`, `DbDataReader`, `DbContext`.
- **Locks**: `SemaphoreSlim`, `Mutex`, `ManualResetEventSlim`.
- **Cryptography**: hash algorithms, symmetric algorithm instances.
- **Process / Thread**: `Process`, `CancellationTokenSource`.
- **Graphics**: `Bitmap`, `Graphics`, `Brush` (GDI handles).

Rule of thumb: if a type opens a file, socket, OS handle, native memory, or holds a lock — it probably implements IDisposable.

---

## Implementing IDisposable

The full **Dispose Pattern** handles both managed and unmanaged resources, plus the finalizer safety net:

```csharp
public class Resource : IDisposable {
    private Stream? _managed;     // managed disposable
    private IntPtr _native;        // unmanaged handle
    private bool _disposed;

    public Resource(string path) {
        _managed = File.OpenRead(path);
        _native = AllocateNativeBuffer(1024);
    }

    public void Dispose() {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing) {
        if (_disposed) return;

        if (disposing) {
            // Caller explicitly disposing — safe to touch managed resources
            _managed?.Dispose();
            _managed = null;
        }

        // Always release unmanaged resources (both Dispose AND finalizer)
        if (_native != IntPtr.Zero) {
            FreeNativeBuffer(_native);
            _native = IntPtr.Zero;
        }

        _disposed = true;
    }

    ~Resource() {
        Dispose(disposing: false);
    }
}
```

The pattern:
1. **`Dispose()`** public method calls `Dispose(true)` and suppresses finalization.
2. **`Dispose(bool disposing)`** virtual method does the actual work. The `disposing` flag distinguishes:
   - `true` — called from `Dispose()`. Both managed AND unmanaged resources to clean up.
   - `false` — called from the finalizer. Only unmanaged (managed disposables may already be finalized — don't touch them).
3. **`~Resource()`** finalizer calls `Dispose(false)` as a safety net for callers who forgot to dispose.
4. **`GC.SuppressFinalize`** removes the object from the finalizer queue (since we already cleaned up).
5. **`_disposed` flag** makes Dispose **idempotent** — calling it twice is safe.

---

## Simplified — when there's no unmanaged

If your type only owns other `IDisposable` instances (no unmanaged handles), you don't need the finalizer:

```csharp
public class SimpleResource : IDisposable {
    private readonly Stream _stream;
    private bool _disposed;

    public SimpleResource(string path) { _stream = File.OpenRead(path); }

    public void Dispose() {
        if (_disposed) return;
        _stream.Dispose();
        _disposed = true;
    }
}
```

No finalizer needed — `_stream` has its own (via `SafeHandle` internally). Forgetting to dispose `SimpleResource` doesn't strand any unmanaged handles directly — eventually the `_stream` finalizer kicks in for its own handle.

Most application classes that aggregate disposables fit this simpler pattern.

---

## SafeHandle — the modern way

For unmanaged handles (file descriptors, OS handles), use `SafeHandle` instead of `IntPtr`. SafeHandle wraps the handle, has its own finalizer (with critical finalization), and ref-counts during P/Invoke.

```csharp
public class MyHandle : SafeHandle {
    public MyHandle() : base(IntPtr.Zero, ownsHandle: true) { }
    public override bool IsInvalid => handle == IntPtr.Zero;
    protected override bool ReleaseHandle() {
        // OS-specific cleanup
        CloseHandle(handle);
        return true;
    }
}
```

When the SafeHandle is disposed or finalized, `ReleaseHandle` runs. The runtime guarantees this even during process tear-down.

Now your class holding SafeHandle doesn't need its own finalizer:

```csharp
public class Modern : IDisposable {
    private readonly MyHandle _handle = new();
    public void Dispose() => _handle.Dispose();
}
```

The SafeHandle takes care of unmanaged cleanup. Much simpler. See [DotNetBook Chapter 14 §02](../../DotNetBook/14-InteropAOT/02-SafeHandle.md) for the deep version.

---

## Idempotent Dispose

`Dispose()` should be **safe to call multiple times** — the second call is a no-op:

```csharp
var r = new Resource(...);
r.Dispose();
r.Dispose();   // OK — no-op
```

Used to be controversial; .NET design guidelines now require idempotency. The `_disposed` flag in the pattern enforces it.

Many BCL types throw `ObjectDisposedException` if you try to use them after Dispose, but the Dispose itself doesn't throw.

---

## Using `using` correctly

### Block scope

```csharp
public string Read(string path) {
    using (var stream = File.OpenRead(path)) {
        using (var reader = new StreamReader(stream)) {
            return reader.ReadToEnd();
        }
    }
}
```

Nested usings — each disposes when its block ends. Inner first (the StreamReader), then the outer (FileStream).

### Declaration form (C# 8+)

```csharp
public string Read(string path) {
    using var stream = File.OpenRead(path);
    using var reader = new StreamReader(stream);
    return reader.ReadToEnd();
}   // both disposed here, in reverse declaration order
```

Cleaner. Avoids nesting indentation.

### Returning a disposable

If you return a disposable from a method, the caller owns it:

```csharp
public Stream Open() => File.OpenRead("data.txt");

// Caller:
using var stream = svc.Open();
```

Don't `using` inside the method if you're returning the resource.

### `using` with null

```csharp
Stream? maybe = condition ? File.OpenRead(path) : null;
using (maybe) {
    if (maybe is not null) maybe.Read(...);
}
```

`using` handles null safely — checks before calling Dispose. No NRE.

---

## Async dispose — `IAsyncDisposable`

For resources whose cleanup involves async work:

```csharp
public async ValueTask DisposeAsync() {
    await _stream.FlushAsync();
    await _stream.DisposeAsync();
}
```

Consumed via `await using`. See [Chapter 08 §08](../08-Concurrency/08-AsyncDisposable.md).

---

## Common bugs

### Returning a disposable from inside a using

```csharp
public Stream BadOpen(string path) {
    using var stream = File.OpenRead(path);   // ⚠ — disposed before return!
    return stream;
}
```

The caller receives a disposed stream. Throw or NRE on first use.

Fix:
```csharp
public Stream Open(string path) {
    return File.OpenRead(path);   // caller owns it
}
```

### Forgetting to dispose

```csharp
var stream = File.OpenRead(path);
stream.Read(...);
// missing Dispose — file handle leaks until GC + finalizer
```

For small one-off code: usually noticed in load testing. For long-running services: file/socket exhaustion.

**Use `using` always**. Roslyn analyzers (CA2000) catch most missing-dispose cases.

### Disposing twice via aliased references

```csharp
var stream = File.OpenRead(path);
var reader = new StreamReader(stream);
stream.Dispose();
reader.Dispose();   // tries to dispose the same stream again
```

This case actually works (StreamReader disposes its base stream by default, and Dispose is idempotent), but it shows the principle: be careful about ownership.

### Throw from Dispose

```csharp
public void Dispose() {
    _stream.Close();   // might throw IOException
}
```

If Dispose throws, the original code's try block has its exception replaced (in `using` block, the dispose exception bubbles up only if the original code didn't already throw). Logging and continuing is usually best:

```csharp
public void Dispose() {
    try { _stream.Close(); }
    catch (Exception ex) { _log.LogWarning(ex, "failed to close stream"); }
}
```

Best practice: **never throw from Dispose**.

### Async method using a disposable across `await`

```csharp
public async Task ProcessAsync() {
    using var stream = File.OpenRead(path);   // sync dispose at end
    await ProcessStreamAsync(stream);
}
```

For Stream specifically, `DisposeAsync` is preferred:

```csharp
await using var stream = File.OpenRead(path);
await ProcessStreamAsync(stream);
```

Awaits the dispose. Properly flushes any pending writes.

---

## Internals — what `using` compiles to

```csharp
using var stream = File.OpenRead("data.txt");
// ... use stream ...
```

becomes roughly:

```csharp
var stream = File.OpenRead("data.txt");
try {
    // ... use stream ...
} finally {
    if (stream != null) ((IDisposable)stream).Dispose();
}
```

For `await using`:

```csharp
await using var stream = File.OpenRead("data.txt");
// ... use stream ...
```

becomes:

```csharp
var stream = File.OpenRead("data.txt");
try {
    // ... use stream ...
} finally {
    if (stream != null) {
        if (stream is IAsyncDisposable iad) await iad.DisposeAsync();
        else stream.Dispose();
    }
}
```

The compiler picks `DisposeAsync` if the type implements `IAsyncDisposable`, else falls back to sync.

---

## When to implement IDisposable

✓ Your class owns an `IDisposable` field (composition).
✓ Your class wraps an unmanaged handle (pair with SafeHandle).
✓ Your class holds a long-lived OS resource (network connection, file lock).
✓ Your class allocates pooled resources that need to be returned.

✗ Your class only owns managed memory — GC handles it.
✗ "I might add unmanaged resources later" — no, add IDisposable then.
✗ Trivial classes — adds noise without value.

---

## Patterns

### Aggregating disposables

```csharp
public class Composite : IDisposable {
    private readonly List<IDisposable> _resources = new();
    public void Add(IDisposable r) => _resources.Add(r);
    public void Dispose() {
        foreach (var r in _resources) {
            try { r.Dispose(); }
            catch (Exception ex) { _log.LogWarning(ex, "dispose failed"); }
        }
    }
}
```

Useful for collecting disposables across many objects.

### Stopwatch-style auto-dispose for tracing

```csharp
public class Operation : IDisposable {
    private readonly string _name;
    private readonly Stopwatch _sw = Stopwatch.StartNew();
    public Operation(string name) { _name = name; }
    public void Dispose() => Console.WriteLine($"{_name}: {_sw.ElapsedMilliseconds} ms");
}

using (new Operation("download")) {
    Download();
}
// Prints "download: 123 ms" at end of block
```

Combine with `using` for scoped timing.

### Defer (Go-style)

```csharp
public class Defer : IDisposable {
    private readonly Action _action;
    public Defer(Action a) { _action = a; }
    public void Dispose() => _action();
}

using (new Defer(() => Console.WriteLine("cleanup"))) {
    Console.WriteLine("work");
}
// "work" then "cleanup"
```

Cleanup that runs at end of scope. Useful for one-off teardown.

---

## Performance

- `using` has near-zero overhead vs manual try/finally — same IL.
- Dispose itself: depends on what it does. Releasing a file handle: ~µs (kernel call).
- For very frequent dispose-create cycles: consider pooling (`ArrayPool` for arrays; `ObjectPool<T>` for custom).

---

## Summary

- IDisposable = deterministic cleanup for non-memory resources.
- Always use `using` (block or declaration form).
- Implement the full Dispose Pattern only for types directly owning unmanaged resources.
- For unmanaged handles, use **SafeHandle** — handles finalization correctly.
- For async cleanup, implement `IAsyncDisposable` and use `await using`.
- Dispose should be idempotent and not throw.
- Don't add IDisposable unnecessarily — only when there's something real to clean up.

→ Next: [04-Finalizers.md](04-Finalizers.md)
