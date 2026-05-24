# Finalizers

## What it is

A **finalizer** (sometimes called a "destructor" in C# syntax — though it's different from C++ destructors) is a method the GC runs **just before reclaiming** an object. Used as a safety net for releasing unmanaged resources when callers forget to `Dispose`.

```csharp
public class Resource {
    private IntPtr _handle;

    public Resource() {
        _handle = AllocateNative();
    }

    ~Resource() {
        if (_handle != IntPtr.Zero) {
            FreeNative(_handle);
            _handle = IntPtr.Zero;
        }
    }
}
```

`~ClassName` is the finalizer syntax. The runtime calls it on a dedicated **finalizer thread** when the object becomes unreachable.

Finalizers are **expensive and unreliable** — use them only as a last resort for unmanaged resources, paired with `IDisposable`. Modern code uses `SafeHandle` and avoids finalizers entirely.

---

## Why they exist

Without finalizers, if a caller forgets to dispose an object holding an OS handle, the handle leaks until the process ends. Finalizers provide a cleanup hook the GC invokes for you.

```csharp
public class File {
    private IntPtr _fd;

    public File(string path) { _fd = open(path); }
    public void Dispose() { if (_fd != 0) { close(_fd); _fd = 0; } GC.SuppressFinalize(this); }
    ~File() { if (_fd != 0) close(_fd); }
}
```

Even if `Dispose` isn't called, the finalizer eventually closes the file descriptor. Defense in depth.

The catch: it's **slow** (finalization queue, two GC cycles), and you don't control **when** it runs. Don't rely on it for any logic that needs to be timely.

---

## The cost of a finalizer

When you allocate an object with a finalizer:

1. The CLR registers it in the **finalization registration list**.
2. When the object becomes unreachable, the GC moves it to the **finalization queue** (does not yet collect it).
3. The dedicated **finalizer thread** runs each queued finalizer.
4. After finalization, the object is back to "unreachable, no longer pending finalization" — the **next** GC reclaims it.

Consequences:
- Finalizable objects survive **two GC cycles** (the one that queues them; the one that finally frees them).
- They typically get promoted to **Gen2** during this process — bumping them into the expensive-to-collect tier.
- The finalizer thread is **single-threaded**. A slow finalizer blocks all subsequent finalizations.
- During app shutdown, finalizers may be skipped (if process kill or timeout).

The cumulative effect: a hot path that allocates many finalizable objects pressures the GC badly. Each one becomes "Gen2 promoted, takes longer to free, slows the finalizer thread."

---

## Always pair with IDisposable

The full pattern from [§03 IDisposable](03-IDisposable.md):

```csharp
public class Resource : IDisposable {
    private IntPtr _native;
    private bool _disposed;

    public Resource() { _native = AllocateNative(); }

    public void Dispose() {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);   // ← finalizer no longer needed
    }

    protected virtual void Dispose(bool disposing) {
        if (_disposed) return;
        if (disposing) {
            // managed cleanup — only safe from explicit Dispose
        }
        if (_native != IntPtr.Zero) {
            FreeNative(_native);
            _native = IntPtr.Zero;
        }
        _disposed = true;
    }

    ~Resource() {
        Dispose(disposing: false);   // finalizer fallback
    }
}
```

Key points:
- `GC.SuppressFinalize(this)` in `Dispose()` — tells the GC "I've cleaned up; don't bother with the finalizer." Avoids the two-cycle overhead.
- `~Resource()` calls `Dispose(false)` — same cleanup, but **don't touch managed disposables** (they may already be finalized by their own finalizers).

---

## Why "don't touch managed disposables" in the finalizer

```csharp
~MyClass() {
    _someManagedStream.Dispose();   // ⚠ might be invalid
}
```

The finalizer runs after the object is unreachable. **Other unreachable objects' finalizers may have already run**. If `_someManagedStream` had its own finalizer (or its inner SafeHandle did), the underlying handle might already be closed.

Worse: managed disposables generally don't expect to be called from the finalizer thread. They might lock, or assume they're being disposed in normal flow.

**Rule**: in `Dispose(false)` (the finalizer path), only release **directly-owned unmanaged resources** (your own IntPtr handles, native memory). Skip managed disposables — their own finalizers will handle them.

---

## SafeHandle — why you should rarely write a finalizer

`SafeHandle` is a BCL type that wraps a native handle and has its own optimized finalization mechanism (critical finalization, ref-counting during P/Invoke).

```csharp
public class MyHandle : SafeHandle {
    public MyHandle() : base(IntPtr.Zero, ownsHandle: true) { }
    public override bool IsInvalid => handle == IntPtr.Zero;
    protected override bool ReleaseHandle() {
        CloseHandle(handle);
        return true;
    }
}

public class Modern : IDisposable {
    private readonly MyHandle _h = new();
    public void Dispose() => _h.Dispose();
}
```

`Modern` has no finalizer. SafeHandle handles the safety net. Cleaner, faster, harder to get wrong.

**For new code: use SafeHandle. Skip writing finalizers.**

---

## `GC.SuppressFinalize`

Tells the GC "this object's finalizer has already done its work (or doesn't need to run)." Removes from the finalization queue:

```csharp
public void Dispose() {
    // do cleanup
    GC.SuppressFinalize(this);   // skip the finalizer
}
```

Without this, `Dispose()` runs cleanup, then the finalizer also runs later — pointless extra work, plus object lives an extra GC cycle.

Always call `GC.SuppressFinalize(this)` from `Dispose()` if your class has a finalizer.

---

## `GC.ReRegisterForFinalize`

Re-arms the finalizer if you decided you want it again:

```csharp
GC.ReRegisterForFinalize(this);
```

Used in **object resurrection** patterns (where you bring an object back from the dead). Almost always a bad idea — see "Resurrection" below.

---

## What you can NOT do in a finalizer

The finalizer runs on a special thread, when the object is otherwise unreachable. Restrictions:

### Don't access other managed objects with finalizers

```csharp
private readonly OtherFinalizable _other;
~Me() {
    _other.DoSomething();   // ⚠ _other may have been finalized already
}
```

Finalization order is **not deterministic**. Another finalizable object the same age might run before or after yours.

### Don't throw

```csharp
~Me() {
    throw new Exception("?");   // ⚠ — in .NET Framework, crashed the process; in modern .NET, swallowed but problematic
}
```

Exceptions in finalizers used to terminate the process. Modern .NET swallows them, but you've lost any error information.

### Don't take a long time

The finalizer thread is sequential. A slow finalizer delays all others. If yours takes 1 second, the next finalizable object waits 1 second.

### Don't block on async / external resources

```csharp
~Me() {
    Task.Delay(1000).Wait();   // ⚠ blocks finalizer thread
}
```

Blocks the whole finalizer queue.

### Don't expect AppDomain unload semantics

During process shutdown, finalizers may not run at all if the process is killed forcibly. Don't rely on finalizers for "must-happen-on-shutdown" logic — use `AppDomain.CurrentDomain.ProcessExit` or `IHostApplicationLifetime.ApplicationStopping` instead.

---

## Resurrection — don't do this

```csharp
public static List<Me> _resurrected = new();

~Me() {
    _resurrected.Add(this);   // ⚠ this is now reachable again
}
```

The object becomes reachable from the static list. The GC won't reclaim it. Its finalizer **won't run again** (unless you re-register).

Resurrection is a rarely-useful trick (for object pooling) that's usually a bug. Modern .NET has `ObjectPool<T>` if you want pooling; don't roll it via resurrection.

---

## CriticalFinalizerObject — guaranteed finalization

`System.Runtime.ConstrainedExecution.CriticalFinalizerObject` is a base class for finalizers that **must** run, even in dire circumstances (out of memory, AppDomain unload, process tear-down).

```csharp
public class MyCritical : CriticalFinalizerObject {
    ~MyCritical() {
        // guaranteed to run
    }
}
```

`SafeHandle` derives from this — that's part of why it's reliable. Used by the runtime; rarely needed in app code.

---

## Internals — how finalization works

Each object has a flag in its header indicating "finalizable." When the object is allocated:

1. CLR adds it to the **finalization registration list** (just tracking what objects have finalizers).

When the object becomes unreachable:

2. GC sees the finalization flag; moves the object to the **finalization queue**.
3. The object is now reachable through the queue — **NOT collected**.

The **finalizer thread** loops, dequeuing objects, calling their finalizers:

4. Run the finalizer (`~ClassName` body).
5. Mark the object as "finalized" — no longer in the queue.

Next GC:

6. Object is unreachable AND finalized → reclaim normally.

This is why finalizable objects survive **two cycles**: one to queue them, one to actually free them.

### Generation promotion

When an object is queued for finalization, it's effectively kept alive. If it was in Gen0, the next GC promotes it to Gen1. If long enough, to Gen2. Most finalizable objects end up in **Gen2** by the time they're truly collected.

This is why hot allocation of finalizable objects is a perf disaster: every one becomes a Gen2 object, even if logically short-lived. Gen2 collections are expensive.

`GC.SuppressFinalize(this)` removes the object from the finalization queue, dodging this entire mess.

---

## Common patterns

### SafeHandle-based design

```csharp
public class File {
    private readonly SafeFileHandle _handle;
    public File(string path) {
        _handle = NativeMethods.CreateFile(path, ...);
    }
    public void Dispose() => _handle.Dispose();
    // No finalizer — SafeFileHandle has its own
}
```

Clean, no finalizer overhead in this class. Always prefer this pattern.

### Defensive cleanup with try/catch in finalizer

```csharp
~Resource() {
    try {
        if (_native != IntPtr.Zero) {
            FreeNative(_native);
            _native = IntPtr.Zero;
        }
    } catch {
        // swallow — finalizer must not throw
    }
}
```

Belt-and-suspenders. The runtime is more forgiving in modern .NET, but explicit swallow is fine.

---

## Common bugs

### Forgetting `GC.SuppressFinalize`

```csharp
public void Dispose() {
    FreeNative();
    // missing GC.SuppressFinalize(this)
}
~MyClass() { FreeNative(); }
```

After Dispose, the finalizer still runs — double-free attempt. The `_disposed` flag in the pattern prevents actual double-free, but the GC overhead is real.

### Touching managed state in finalizer

```csharp
~MyClass() {
    _logger.Log("disposing");   // ⚠ _logger might be unreachable / disposed
}
```

Don't. The finalizer path should only release directly-owned unmanaged resources.

### Finalizer that throws

```csharp
~MyClass() {
    if (someInvariantViolated) throw new InvalidOperationException();
}
```

Exceptions from finalizers are swallowed in modern .NET, but problematic in .NET Framework (process crashed). And you lose all error context. Never throw.

### Slow finalizer

```csharp
~Resource() {
    Thread.Sleep(1000);
    FreeNative();
}
```

Blocks the entire finalizer queue. Other finalizable objects wait. Fix the cleanup to be fast.

### Relying on finalizer ordering

```csharp
class A { ~A() { Console.WriteLine("A"); } }
class B { public A InnerA; ~B() { Console.WriteLine("B"); } }
```

If both are unreachable, A might run before B (if A is processed first by the finalizer thread). B's finalizer can't safely touch InnerA. Order isn't guaranteed.

---

## When you NEED a finalizer

Almost never in application code. The cases:
- You directly hold an unmanaged handle (`IntPtr` to native memory or OS resource) without wrapping in SafeHandle.
- You're writing a SafeHandle subclass (use SafeHandle's `ReleaseHandle` instead).
- Legacy code requirements.

For new code: 99% of the time, **use SafeHandle**. If you can't (for some specific interop reason), then a finalizer + dispose pattern.

---

## Performance summary

- Allocating a finalizable object: same as any allocation (+ adding to finalization registration).
- Becoming unreachable: + moves to finalization queue (delays actual reclaim).
- Finalizer thread time: ~µs per finalizer + your finalizer's body.
- Net per finalizable object: 2 GC cycles of "alive" time.

For 10,000 finalizable allocations / sec, the finalizer thread becomes saturated, queue grows. Memory pressure ensues. Avoid finalizers in hot paths.

---

## Summary

- A finalizer is a safety net for unmanaged resources when callers forget Dispose.
- Always pair with IDisposable + `GC.SuppressFinalize(this)` in Dispose.
- Finalizers are slow (two GC cycles, sequential thread, Gen2 promotion).
- Modern best practice: use **SafeHandle** to encapsulate the handle + finalization. Avoid writing finalizers directly.
- Don't touch managed state from a finalizer; don't throw; be fast.
- For "must run on shutdown" logic, use ProcessExit / hosted service lifecycle, not finalizers.

→ Next: [05-Span.md](05-Span.md)
