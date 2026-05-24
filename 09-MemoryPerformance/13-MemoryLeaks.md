# Memory Leaks

## What it is

A **memory leak** in managed code happens when objects you no longer need are still reachable from a GC root — so the GC can't reclaim them. Unlike C/C++ leaks (forgotten `delete`), managed leaks come from accidentally **keeping references alive**.

```csharp
// Static cache that never evicts
public static class Cache {
    private static Dictionary<int, Data> _items = new();
    public static void Add(int id, Data d) => _items[id] = d;
}

// Over time, _items grows forever — memory leak
```

Symptoms:
- Process memory keeps climbing.
- Gen2 heap size grows over time.
- Eventually: `OutOfMemoryException` or container OOM-kill.

This file is the diagnostic playbook for finding and fixing them.

---

## The big sources of leaks

### 1. Static caches without bounds

```csharp
public static class Cache {
    private static readonly Dictionary<string, Data> _items = new();
}
```

Statics are GC roots. Anything they reference (and anything THAT references, transitively) is alive forever.

**Fix**: bounded cache with eviction.

```csharp
private static readonly MemoryCache _items = new(new MemoryCacheOptions {
    SizeLimit = 1000
});
```

`Microsoft.Extensions.Caching.Memory.MemoryCache` evicts via LRU when size limit is hit. Bounded memory.

Or use `IDistributedCache` (Redis etc.) for shared / out-of-process caching.

### 2. Event handlers (subscriber leak)

```csharp
class Subscriber {
    public Subscriber(Publisher pub) {
        pub.Event += this.OnEvent;
    }
    void OnEvent(object? sender, EventArgs e) { /* ... */ }
}
```

`pub.Event` (the publisher's delegate) holds a reference to `this`. As long as Publisher is alive, every Subscriber that subscribed is alive.

If Publisher is long-lived and Subscribers are short-lived → **leak**: subscribers accumulate.

**Fix**: unsubscribe in Dispose:

```csharp
class Subscriber : IDisposable {
    private readonly Publisher _pub;
    public Subscriber(Publisher pub) {
        _pub = pub;
        _pub.Event += OnEvent;
    }
    public void Dispose() => _pub.Event -= OnEvent;
    void OnEvent(object? s, EventArgs e) { /* ... */ }
}
```

`using` ensures Dispose. Subscription cleanly released.

For ASP.NET Core: scoped services that subscribe to singleton events leak the scoped instance until the singleton dies. Always unsubscribe.

### 3. Timer callbacks

```csharp
class Worker {
    private readonly Timer _timer = new(OnTick, null, 0, 1000);
    void OnTick(object? state) { /* ... */ }
}
```

The `Timer` holds a reference to its callback delegate. The delegate references `this` (via the captured method). Worker stays alive as long as Timer fires.

**Fix**: implement IDisposable, dispose the Timer:

```csharp
public void Dispose() => _timer.Dispose();
```

### 4. Lambda captures

```csharp
public class Service {
    public byte[] BigBuffer = new byte[100_000_000];   // 100 MB

    public Func<int> MakeFunc() {
        int local = 42;
        return () => local + 1;   // captures `local` — and IMPLICITLY `this`!
    }
}
```

Even though the lambda only uses `local`, the closure class may hold a reference to `this` (the captured locals are usually fields of a closure class; the class implicitly holds `this`).

Result: returning a lambda from a Service method keeps the Service (and its 100 MB buffer) alive as long as the lambda is referenced.

**Fix**:
- Copy needed values to locals BEFORE the lambda; avoid capturing `this`.
- Or use a static lambda where possible.

```csharp
public Func<int> MakeFunc() {
    int local = 42;
    int snapshot = 0;   // don't reference any field
    return () => snapshot + local;
}
```

### 5. Long-lived references in collections

```csharp
public class History {
    private readonly List<Event> _events = new();
    public void Record(Event e) => _events.Add(e);
}
```

`_events` grows forever. Each Event (and anything it references) stays alive.

**Fix**: bound the history (LRU cache, ring buffer, drop old entries).

### 6. Thread-static caches

```csharp
[ThreadStatic]
private static Dictionary<int, Data>? _cache;
```

Per-thread state. For long-lived threads (e.g., thread pool workers), the cache lives as long as the thread.

The thread pool reuses threads, so `_cache` stays around AppDomain-long. Same leak class as static.

### 7. Pinned objects

```csharp
GCHandle.Alloc(obj, GCHandleType.Pinned);
```

A pinned object can't be moved or collected until the handle is freed. Forgetting `handle.Free()` leaks the object.

Pre-.NET 5, also fragmented the heap. .NET 5 introduced POH (Pinned Object Heap) to isolate pins.

### 8. Native interop without releasing

```csharp
IntPtr ptr = Marshal.AllocHGlobal(1024);
// forgot Marshal.FreeHGlobal(ptr)
```

Native memory isn't GC-tracked. Leaks forever.

**Fix**: `SafeHandle` or `using (var b = new NativeBuffer(1024)) { ... }`. Always pair Alloc with Free.

### 9. Disposables not disposed

```csharp
var stream = File.OpenRead(path);
// forgot to Dispose
```

The OS handle stays open. Sometimes the GC + finalizer cleans up eventually, but in high-frequency code, the file descriptor table runs out.

Same for `DbConnection`, `HttpResponseMessage`, `CancellationTokenSource`.

**Fix**: `using` everywhere.

### 10. Long-running tasks holding captures

```csharp
var bigData = new byte[100_000_000];
_ = Task.Run(async () => {
    await Task.Delay(TimeSpan.FromHours(1));   // task lives 1 hour
    Process(bigData);                            // captured
});
```

The Task captures `bigData`. The Task is referenced (by the thread pool / runtime) for the hour it lives. `bigData` stays alive.

**Fix**: copy what you need, avoid capturing big buffers.

---

## Diagnostic tools

### `dotnet-counters` — live monitoring

```bash
dotnet-counters monitor -p <pid> --counters System.Runtime
```

Watch:
- `gc-heap-size` — total heap size. Growing over time = leak.
- `gen-2-size` — long-lived objects. Growing = leak.
- `% time in gc` — high = pressure.

First step in diagnosing.

### `dotnet-gcdump` — heap snapshot

```bash
dotnet-gcdump collect -p <pid>
```

Captures a heap snapshot. Open in PerfView or dotMemory.

Take two dumps an hour apart. **Diff them**: objects that appeared and stayed are likely leaks.

### `dotnet-dump` — full process dump

```bash
dotnet-dump collect -p <pid>
```

Full memory dump. Open with `dotnet-dump analyze dumpfile`:

```
> dumpheap -stat            # objects by type, sorted by total size
> dumpheap -type byte[]      # all byte arrays
> gcroot <addr>               # who holds this object alive?
```

`gcroot` is the killer feature: shows the chain of references from a GC root to your object. Tells you exactly what's preventing collection.

### PerfView (Windows)

The Swiss army knife for .NET diagnostics. Heap snapshot diffing, allocation traces, GC analysis. Free.

### JetBrains dotMemory

Commercial profiler with a great UI. Heap snapshots, allocation traces, retention chains. Easy to identify leak sources.

---

## The diagnostic playbook

When you suspect a leak:

1. **Confirm it's a leak** — `dotnet-counters` shows growing heap over hours, even after sustained idle.
2. **Take baseline + leak snapshots** — `dotnet-gcdump` at startup, then again after the leak grew.
3. **Diff snapshots** — what types appeared and stayed?
4. **Investigate top retainers** — sort by retained size.
5. **`gcroot`** specific instances — find the reference chain holding them.
6. **Identify the root** — usually a static field, event handler, timer, or long-lived collection.
7. **Fix** — bound the collection, unsubscribe, dispose.
8. **Verify** — repeat the snapshot; the type should not appear (or its count should be stable).

---

## Common pattern: event leak

Diagnostic chain you'd see:

```
Subscriber instance → __EventHandler<EventArgs> → Publisher.Event delegate → Publisher
                                                ↑
                                                Publisher is a GC root (static or long-lived).
                                                Subscriber lives as long as Publisher does.
```

`gcroot` shows: Subscriber → root reaches via Publisher's event.

**Fix**: unsubscribe in Subscriber.Dispose. Subsequent snapshots show no growth.

---

## Common pattern: closure leak

Diagnostic chain:

```
SomeClass.<>c__DisplayClass0 (closure) → SomeClass.this → ... lots of fields including BigBuffer
                                       ↑
                                       Held by some Func<T> that someone stored.
```

The closure captured `this` because the lambda referenced an instance field. The lambda is stored somewhere (a list, an event, a static). Result: SomeClass + BigBuffer pinned alive.

**Fix**: copy needed values to locals before lambda; don't reference `this` fields directly.

---

## Common pattern: cache leak

Diagnostic chain:

```
[static field] Cache._items → Dictionary<string, Data> → entries → Data → ...
```

Dictionary grows forever as new keys come in. Each entry is a GC root via the static field.

**Fix**: MemoryCache with SizeLimit, or LRU eviction, or distributed cache.

---

## Weak references

For some patterns, you genuinely want to "hold a reference if the object's alive elsewhere, but don't keep it alive":

```csharp
var weak = new WeakReference<MyObj>(myObj);

// Later
if (weak.TryGetTarget(out var obj)) {
    // obj is alive — use it
}
// else — GC reclaimed it; that's fine
```

`WeakReference<T>` doesn't count as a strong reference. The GC can collect the target whenever.

Used in:
- Object caches where you'd rather lose a cached object than block its GC.
- Event subscription patterns (WeakEventManager in WPF).
- Hidden self-references in pub-sub.

For most caches, **bounded MemoryCache is simpler and more predictable**. Weak references have edge cases (GC timing, partial finalization).

---

## Common bugs and patterns

### LoH growth from large arrays not pooled

```csharp
for (int i = 0; i < 100_000; i++) {
    byte[] buf = new byte[100_000];   // LOH allocation per iteration
    Process(buf);
}
// LOH grows; only Gen2 collections reclaim.
```

LOH isn't compacted by default. Fragmentation possible.

**Fix**: `ArrayPool<byte>.Shared.Rent(100_000)` instead.

### HttpClient leak

```csharp
public async Task<string> Get(string url) {
    using var http = new HttpClient();   // ⚠ — socket leak per call
    return await http.GetStringAsync(url);
}
```

`HttpClient.Dispose()` doesn't immediately close TCP connections — they linger in TIME_WAIT. Many short-lived HttpClients → socket exhaustion.

**Fix**: inject `IHttpClientFactory` (ASP.NET Core). Single HttpClient or pooled handler.

### Forgotten CancellationTokenSource

```csharp
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
await DoAsync(cts.Token);
// forgot to dispose cts
```

CTS has a Timer (for CancelAfter). Without dispose, the Timer keeps the CTS alive. Many CTS instances = many timers = memory + scheduler overhead.

**Fix**: `using var cts = new ...`.

---

## When to think about leaks

- Production services running for days/weeks — leaks compound.
- High-throughput code creating lots of objects.
- After a feature ships that added new caching / events / disposables.
- When customer reports OOM or container restarts.

For short-lived tools (CLIs, batch jobs): leaks usually don't matter — process exits and reclaims everything.

---

## Performance vs Memory tradeoffs

Sometimes "leaks" are intentional:

- A `static readonly` config that loads at startup and lives forever.
- An `ImmutableDictionary` snapshot for fast lookups.

These aren't leaks if their size is bounded and known. They're features.

Leak = unbounded growth. Long-lived but bounded data = fine.

---

## Cheat sheet for prevention

- Bounded caches with eviction (MemoryCache, FrozenDictionary, LRU).
- Always dispose disposables (using).
- Always unsubscribe from events on dispose.
- Always dispose timers.
- Pin only when needed; free pins promptly.
- Use SafeHandle for unmanaged handles.
- Avoid capturing `this` in long-lived lambdas.
- Use `IHttpClientFactory` for HTTP.
- Inject `IMemoryCache` instead of static dictionaries.
- Monitor heap size in production.

---

## Summary

- Managed memory "leaks" are accidentally-retained references.
- Top causes: static caches without bounds, event handler subscription leaks, timer captures, lambda captures of `this`, unbounded collections.
- Diagnose with `dotnet-counters`, `dotnet-gcdump`, dotMemory, PerfView.
- `gcroot` shows what's pinning the object alive.
- Fixes: bound caches, unsubscribe, dispose, use weak references where appropriate.
- For high-throughput code: profile under load over time, not just in dev.

→ Continue to: [Questions.md](Questions.md)
