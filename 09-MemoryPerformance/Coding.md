# Chapter 09 — Coding Problems

> 15 hands-on memory & performance problems. Pair with BenchmarkDotNet when you want hard numbers.

---

## Problem 1 — Write Dispose Pattern correctly

Implement `Resource` that owns a managed `Stream` and an unmanaged `IntPtr`. Apply the full Dispose Pattern.

<details><summary>Solution</summary>

```csharp
public class Resource : IDisposable {
    private Stream? _stream;
    private IntPtr _native;
    private bool _disposed;

    public Resource(string path) {
        _stream = File.OpenRead(path);
        _native = Marshal.AllocHGlobal(1024);
    }

    public void Dispose() {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing) {
        if (_disposed) return;
        if (disposing) {
            _stream?.Dispose();
            _stream = null;
        }
        if (_native != IntPtr.Zero) {
            Marshal.FreeHGlobal(_native);
            _native = IntPtr.Zero;
        }
        _disposed = true;
    }

    ~Resource() => Dispose(disposing: false);
}
```

Modern code with SafeHandle would skip the finalizer. But this is the textbook pattern.

</details>

---

## Problem 2 — Zero-allocation number formatting

Write a method `FormatPair(int a, int b)` that returns `"$a,$b"` with **zero allocations** beyond the final string.

<details><summary>Solution</summary>

```csharp
public string FormatPair(int a, int b) {
    Span<char> buf = stackalloc char[32];
    int pos = 0;
    a.TryFormat(buf[pos..], out int wa);
    pos += wa;
    buf[pos++] = ',';
    b.TryFormat(buf[pos..], out int wb);
    pos += wb;
    return new string(buf[..pos]);
}
```

`stackalloc` for the buffer (no heap). `TryFormat` writes the digits in place. One final string allocation.

Compare to `$"{a},{b}"` which (in modern C#) may also avoid intermediate allocations via interpolated string handlers — interpolation is often just as fast.

</details>

---

## Problem 3 — Slice a string without allocation

Find the substring between '<' and '>' in input, returning a ReadOnlySpan<char> view.

<details><summary>Solution</summary>

```csharp
public ReadOnlySpan<char> Extract(string input) {
    int start = input.IndexOf('<');
    int end = input.IndexOf('>');
    if (start < 0 || end < 0 || start >= end) return ReadOnlySpan<char>.Empty;
    return input.AsSpan(start + 1, end - start - 1);
}

var s = Extract("hello <world> goodbye");
Console.WriteLine(s.ToString());   // "world"
```

`AsSpan` is free — just a struct construction. The Span is a view into the original string. No allocation.

Note `.ToString()` at the end IS an allocation if you actually need a string. To work fully zero-alloc, keep using the span downstream.

</details>

---

## Problem 4 — ArrayPool wrapper

Implement a `PooledArray<T>` struct that rents on construction, returns on Dispose, exposes a `Span<T>` for use.

<details><summary>Solution</summary>

```csharp
public struct PooledArray<T> : IDisposable {
    private T[]? _array;
    private readonly int _size;

    public PooledArray(int size) {
        _array = ArrayPool<T>.Shared.Rent(size);
        _size = size;
    }

    public Span<T> Span => _array.AsSpan(0, _size);

    public void Dispose() {
        if (_array is not null) {
            ArrayPool<T>.Shared.Return(_array);
            _array = null;
        }
    }
}

// Usage
using var p = new PooledArray<byte>(4096);
stream.Read(p.Span);
Process(p.Span);
```

`using` calls Dispose, which returns the array. Caller never sees the raw array — just the Span sized to what they asked for.

</details>

---

## Problem 5 — Find the memory leak

```csharp
public class EventBus {
    public event Action<string>? OnMessage;

    public void Publish(string msg) => OnMessage?.Invoke(msg);
}

public class Listener {
    public Listener(EventBus bus) {
        bus.OnMessage += HandleMessage;
    }

    void HandleMessage(string msg) { /* ... */ }
}

// Long-running app
var bus = new EventBus();
for (int i = 0; i < 1_000_000; i++) {
    var listener = new Listener(bus);
    // listener "goes out of scope" — does it get GC'd?
}
```

<details><summary>Answer</summary>

**No, Listener instances are NOT GC'd.** The EventBus has a delegate referencing each Listener (via `HandleMessage`). EventBus → delegate → Listener. EventBus is long-lived → all Listeners stay alive.

Result: 1M Listeners pinned, memory grows linearly.

**Fix**: implement IDisposable on Listener, unsubscribe:

```csharp
public class Listener : IDisposable {
    private readonly EventBus _bus;
    public Listener(EventBus bus) {
        _bus = bus;
        _bus.OnMessage += HandleMessage;
    }
    public void Dispose() => _bus.OnMessage -= HandleMessage;
    void HandleMessage(string msg) { }
}

using var listener = new Listener(bus);
// Disposed cleanly at end of using
```

Or use a weak event pattern (WeakEventManager in WPF, custom for other contexts).

</details>

---

## Problem 6 — Spot the LOH allocation

```csharp
for (int i = 0; i < 1000; i++) {
    var data = new byte[100_000];   // ⚠
    Process(data);
}
```

What's wrong? How to fix?

<details><summary>Answer</summary>

`new byte[100_000]` is ≥85,000 bytes → goes to the **LOH**. LOH isn't compacted by default. 1000 iterations → 100 MB allocated to LOH, which fragments under varying sizes.

Each LOH allocation requires Gen2 GC for reclaim — expensive.

**Fix**: ArrayPool.

```csharp
for (int i = 0; i < 1000; i++) {
    byte[] data = ArrayPool<byte>.Shared.Rent(100_000);
    try {
        Process(data, 100_000);   // pass requested size!
    } finally {
        ArrayPool<byte>.Shared.Return(data);
    }
}
```

The pool reuses the same large arrays. No LOH churn.

Note: ArrayPool's "Shared" pool typically supports arrays up to ~1 MB by default. Beyond that, falls back to direct allocation.

</details>

---

## Problem 7 — Implement a bounded cache

Build a cache that evicts the oldest entry when full (FIFO eviction). 100 max items.

<details><summary>Solution</summary>

```csharp
public class FifoCache<TKey, TValue> where TKey : notnull {
    private readonly int _capacity;
    private readonly Dictionary<TKey, LinkedListNode<(TKey K, TValue V)>> _map = new();
    private readonly LinkedList<(TKey K, TValue V)> _order = new();

    public FifoCache(int capacity) { _capacity = capacity; }

    public void Set(TKey key, TValue value) {
        if (_map.ContainsKey(key)) {
            var existing = _map[key];
            existing.Value = (key, value);
            return;
        }
        if (_map.Count == _capacity) {
            var oldest = _order.First!;
            _order.RemoveFirst();
            _map.Remove(oldest.Value.K);
        }
        var node = _order.AddLast((key, value));
        _map[key] = node;
    }

    public bool TryGet(TKey key, out TValue value) {
        if (_map.TryGetValue(key, out var node)) {
            value = node.Value.V;
            return true;
        }
        value = default!;
        return false;
    }
}
```

For LRU eviction (use the entry on Get → move to back), see CSharpBook Chapter 07 §03's LRU pattern.

For production: `MemoryCache` with `SizeLimit` does this automatically.

</details>

---

## Problem 8 — Generic constraint for unmanaged

Write a generic method `SumBytes<T>` that takes any `unmanaged` value type and treats its memory as a sequence of bytes for hashing.

<details><summary>Solution</summary>

```csharp
public static int SumBytes<T>(ref T value) where T : unmanaged {
    Span<byte> bytes = MemoryMarshal.AsBytes(MemoryMarshal.CreateSpan(ref value, 1));
    int sum = 0;
    foreach (var b in bytes) sum += b;
    return sum;
}

// Use
var p = new Point(3, 4);
int hash = SumBytes(ref p);
```

`MemoryMarshal.CreateSpan` makes a Span of 1 T pointing at the value. `MemoryMarshal.AsBytes` reinterprets it as bytes. Sum the bytes. No allocation, no unsafe code.

This kind of reinterpret-cast pattern is common in serialization, hashing, struct-to-bytes conversion. Safe-ish (the `unmanaged` constraint ensures no GC-trackable references inside T).

</details>

---

## Problem 9 — Implement a basic SafeHandle

Write a `MyHandle : SafeHandle` that wraps a fake "OS handle" (an `IntPtr` from Marshal.AllocHGlobal).

<details><summary>Solution</summary>

```csharp
public sealed class MyHandle : SafeHandle {
    public MyHandle() : base(IntPtr.Zero, ownsHandle: true) { }

    public static MyHandle Allocate(int size) {
        var h = new MyHandle();
        h.SetHandle(Marshal.AllocHGlobal(size));
        return h;
    }

    public override bool IsInvalid => handle == IntPtr.Zero;

    protected override bool ReleaseHandle() {
        if (handle != IntPtr.Zero) {
            Marshal.FreeHGlobal(handle);
        }
        return true;
    }
}

// Use
using var h = MyHandle.Allocate(1024);
// Use h.DangerousGetHandle() for native interop
// At end of using: ReleaseHandle called automatically
```

`ReleaseHandle` is the OS-specific cleanup. The SafeHandle's own finalizer ensures cleanup even if Dispose is forgotten. No need for explicit finalizer in your class.

</details>

---

## Problem 10 — Diagnose with gcroot (theoretical)

A production heap dump shows 10K `Customer` instances retained. Walk through how you'd find the cause.

<details><summary>Solution</summary>

```
$ dotnet-dump analyze dumpfile.dmp

> dumpheap -type Customer -stat
              MT    Count   TotalSize Class Name
00007f8800a3b900   10000   3,520,000 MyApp.Customer

> dumpheap -type Customer -short
0x7f8801a3b900
0x7f8801a3c100
...  (10K addresses)

> gcroot 0x7f8801a3b900
HandleTable:
    00007f8800a00100 (Strong)
          -> 00007f8800c00500 MyApp.Cache
          -> 00007f8800c00b00 System.Collections.Generic.Dictionary<...>
          -> 00007f8800c01000 System.Collections.Generic.Dictionary<...>+Entry[]
          -> 00007f8801a3b900 MyApp.Customer
```

The chain shows: a static field `MyApp.Cache` holds a Dictionary that holds an array of entries, each referencing a Customer. 10K customers because the dictionary has 10K entries.

**Conclusion**: `Cache` is a static dictionary growing unboundedly. Replace with `MemoryCache` (bounded) or implement eviction.

</details>

---

## Problem 11 — Reuse a buffer across async

Implement `ReadFileAsync` that reads a file 4 KB at a time, using a pooled buffer.

<details><summary>Solution</summary>

```csharp
public async Task<byte[]> ReadFileAsync(string path, CancellationToken ct = default) {
    using var fileStream = File.OpenRead(path);
    using var ms = new MemoryStream();

    using var owner = MemoryPool<byte>.Shared.Rent(4096);
    Memory<byte> buffer = owner.Memory;

    int read;
    while ((read = await fileStream.ReadAsync(buffer, ct)) > 0) {
        await ms.WriteAsync(buffer[..read], ct);
    }
    return ms.ToArray();
}
```

`MemoryPool` for async-friendly buffer rental. `Memory<byte>` crosses awaits safely. The buffer is reused across loop iterations; one rental per call.

</details>

---

## Problem 12 — Show that `volatile` matters

Write a producer-consumer test that fails without `volatile` (on certain hardware), succeeds with it.

<details><summary>Sketch</summary>

```csharp
class Producer {
    private bool _ready;       // try also: volatile bool _ready;
    private int _value;

    public void Produce() {
        _value = 42;
        _ready = true;          // hope reader sees these in this order
    }

    public int Consume() {
        while (!_ready) ;      // spin
        return _value;
    }
}

// Run on two threads:
var p = new Producer();
var t1 = Task.Run(() => p.Produce());
var t2 = Task.Run(() => Console.WriteLine(p.Consume()));
```

On x64 (strong memory model), this almost always works correctly even without volatile (the writes happen in order; CPU doesn't reorder them).

On ARM (weak memory), the writer's `_ready = true` can be observed BEFORE `_value = 42` becomes visible. Reader sees `_value == 0` momentarily.

With `volatile bool _ready`: the write to `_ready` includes a release barrier — all prior writes (including `_value`) become visible before `_ready` does. Safe.

In practice for portability: always use proper synchronization (`Interlocked`, `lock`, or `volatile`). Don't rely on x64's strong memory model.

</details>

---

## Problem 13 — Avoid closure allocation

```csharp
public List<int> FindAll(List<int> source, int threshold) {
    return source.Where(x => x > threshold).ToList();
}
```

How could you avoid the closure allocation for the predicate?

<details><summary>Solution</summary>

`x => x > threshold` captures `threshold` → creates a closure object per call.

For ultra-hot code, eliminate the capture:

```csharp
// Manual filtering — no lambda
public List<int> FindAll(List<int> source, int threshold) {
    var result = new List<int>(source.Count / 2);
    for (int i = 0; i < source.Count; i++) {
        if (source[i] > threshold) result.Add(source[i]);
    }
    return result;
}
```

Or pass state explicitly:

```csharp
public static IEnumerable<T> Filter<T, TState>(this IEnumerable<T> source, TState state, Func<T, TState, bool> pred) {
    foreach (var x in source) if (pred(x, state)) yield return x;
}

source.Filter(threshold, static (x, t) => x > t).ToList();
// static lambda + threading state through avoids closure
```

For typical code, the closure allocation is invisible. For tight inner loops, worth measuring.

.NET 10's escape analysis may stack-allocate the closure if it doesn't escape — sometimes makes manual elimination unnecessary.

</details>

---

## Problem 14 — Profile this with BenchmarkDotNet

Set up a benchmark comparing:
1. `string.Join(",", numbers)` for `int[1000]`.
2. Manual StringBuilder + comma.
3. Manual stackalloc + TryFormat.

<details><summary>Sketch</summary>

```csharp
[MemoryDiagnoser]
public class JoinBench {
    private readonly int[] _data = Enumerable.Range(0, 1000).ToArray();

    [Benchmark]
    public string StringJoin() => string.Join(",", _data);

    [Benchmark]
    public string StringBuilder() {
        var sb = new StringBuilder();
        for (int i = 0; i < _data.Length; i++) {
            if (i > 0) sb.Append(',');
            sb.Append(_data[i]);
        }
        return sb.ToString();
    }

    [Benchmark]
    public string Stackalloc() {
        Span<char> buf = stackalloc char[8192];   // enough for 1000 ints
        int pos = 0;
        for (int i = 0; i < _data.Length; i++) {
            if (i > 0) buf[pos++] = ',';
            _data[i].TryFormat(buf[pos..], out int w);
            pos += w;
        }
        return new string(buf[..pos]);
    }
}
```

Typical numbers (intel x64):
- StringJoin: ~30 µs, 1 allocation (the result).
- StringBuilder: ~30-50 µs, 1-2 allocations.
- Stackalloc: ~15-20 µs, 1 allocation.

`string.Join` is internally optimized — competitive. Stackalloc is fastest for known-size cases. For unknown sizes, ArrayPool-backed builder.

</details>

---

## Problem 15 — Implement an object pool

Build `ObjectPool<T>` that rents and returns instances. Use ConcurrentBag for thread-safety.

<details><summary>Solution</summary>

```csharp
public class ObjectPool<T> where T : class, new() {
    private readonly ConcurrentBag<T> _items = new();
    private readonly Func<T>? _factory;

    public ObjectPool(Func<T>? factory = null) { _factory = factory; }

    public T Rent() {
        if (_items.TryTake(out var item)) return item;
        return _factory?.Invoke() ?? new T();
    }

    public void Return(T item) {
        if (item is IResettable r) r.Reset();   // optional reset hook
        _items.Add(item);
    }
}

public interface IResettable {
    void Reset();
}

// Use
var pool = new ObjectPool<StringBuilder>(() => new StringBuilder());
var sb = pool.Rent();
sb.Append("hello");
var result = sb.ToString();
sb.Clear();   // or implement IResettable
pool.Return(sb);
```

Production: `Microsoft.Extensions.ObjectPool` provides this and more (e.g., bounded pool, ASP.NET Core integration). For per-thread pooling, `[ThreadStatic]` field with manual management.

</details>

---

## Summary

You've now drilled:
- Stack vs heap mental model + JIT realities.
- GC generations, DATAS, LOH/POH.
- IDisposable + Dispose pattern.
- Finalizers and SafeHandle.
- Span / Memory / stackalloc / ArrayPool.
- ref features: ref struct, ref locals, in / ref readonly.
- Unsafe code (when justified) + safer alternatives.
- Escape analysis (.NET 10).
- String interning.
- Memory leak diagnosis with dotnet-counters / gcdump / gcroot.

This is the foundation of high-performance .NET. Internalize it.

→ [Chapter 10 — Advanced Language Features](../10-AdvancedLanguage/README.md)
