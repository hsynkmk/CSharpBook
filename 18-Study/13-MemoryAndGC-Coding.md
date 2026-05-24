# 13 — Memory & GC — Coding Questions ⭐

> Find the leak / predict behavior. (Concepts: [13-MemoryAndGC.md](13-MemoryAndGC.md))

---

### Q1 — Find the leak (event handler)
```csharp
class Worker {
    public Worker(EventBus bus) => bus.OnMessage += Handle;   // bus is a singleton
    void Handle(object? s, MsgArgs e) { }
}
// thousands of Workers created and discarded over time
```
<details><summary>Answer</summary>

**Leak.** The long-lived `bus` holds each `Worker` via the event delegate → Workers are never GC'd. **Fix:** unsubscribe (`bus.OnMessage -= Handle`) in `Dispose`/`IDisposable`, or use weak event handlers. The #1 .NET leak.
</details>

---

### Q2 — Static cache leak
```csharp
static class Cache {
    static readonly Dictionary<string, byte[]> _items = new();
    public static void Add(string key, byte[] data) => _items[key] = data;
}
```
<details><summary>Answer</summary>

**Unbounded static cache = leak.** Everything reachable from a `static` lives for the app's lifetime; `_items` only grows. **Fix:** bound it (LRU / size limit), use `MemoryCache` with expiration, or evict on write. ([18-Caching-Resilience-Http.md](18-Caching-Resilience-Http.md))
</details>

---

### Q3 — Timer keeps object alive
```csharp
class Poller {
    Timer _t;
    public Poller() => _t = new Timer(_ => Poll(), null, 0, 1000);
    void Poll() { }
}
// Poller reference dropped, but Poll keeps running. Why?
```
<details><summary>Answer</summary>

The `Timer` (rooted by the runtime) holds the callback → keeps the `Poller` alive even after you drop your reference → it never gets collected and keeps polling. **Fix:** implement `IDisposable` and `_t.Dispose()` to stop the timer and release the object.
</details>

---

### Q4 — Does Dispose free memory?
```csharp
var stream = new FileStream("a.txt", FileMode.Open);
stream.Dispose();
// is the managed FileStream object's memory freed now?
```
<details><summary>Answer</summary>

**No** — `Dispose` releases the **unmanaged resource** (the OS file handle) deterministically, but the managed `FileStream` *object's* memory is reclaimed later by the **GC** when unreachable. Dispose ≠ free memory; it = release external resources promptly.
</details>

---

### Q5 — using and exceptions
```csharp
using (var conn = new SqlConnection(cs)) {
    conn.Open();
    throw new Exception("boom");
}   // is conn disposed?
```
<details><summary>Answer</summary>

**Yes** — `using` compiles to try/finally, so `conn.Dispose()` runs even when the body throws. That's the whole point of `using` over manual `Dispose`.
</details>

---

### Q6 — LOH allocation
```csharp
var small = new byte[80_000];    // (a)
var big   = new byte[90_000];    // (b)
```
<details><summary>Answer</summary>

(a) goes on the **normal (Gen 0) heap**; (b) ≥ 85,000 bytes → goes on the **Large Object Heap (LOH)** — collected with Gen 2, **not compacted** by default → fragmentation risk. Reuse large buffers with `ArrayPool<byte>` to avoid churning the LOH.
</details>

---

### Q7 — Which generation survives?
```csharp
var obj = new byte[100];
GC.Collect();
Console.WriteLine(GC.GetGeneration(obj));   // (still referenced)
```
<details><summary>Answer</summary>

**`1`** (or higher) — `obj` survived a Gen 0 collection (it's still referenced), so it was **promoted** to Gen 1. Survivors get promoted up the generations; long-lived objects reach Gen 2. (Don't call `GC.Collect()` in real code.)
</details>

---

### Q8 — Span: zero-allocation slicing
```csharp
string csv = "12,34,56";
ReadOnlySpan<char> span = csv;
var first = span.Slice(0, 2);
Console.WriteLine(int.Parse(first));
```
<details><summary>Answer</summary>

**`12`** — `Slice` creates a **view** over the existing string memory with **no allocation** (no substring created). `Span<T>`/`ReadOnlySpan<T>` are the go-to for parsing/slicing without GC pressure. (But `Span` is stack-only — can't be a field or used across `await`.)
</details>

---

### Q9 — SuppressFinalize
```csharp
class Res : IDisposable {
    ~Res() => Cleanup();
    public void Dispose() { Cleanup(); /* ??? */ }
    void Cleanup() { }
}
```
<details><summary>Answer</summary>

Missing **`GC.SuppressFinalize(this);`** in `Dispose`. Without it, even after disposing, the object still goes on the **finalization queue** → survives an extra GC cycle (its finalizer runs needlessly). Add `GC.SuppressFinalize(this)` after cleanup in `Dispose`.
</details>

---

### Q10 — Closure capturing a large object (senior)
```csharp
byte[] huge = new byte[100_000_000];   // 100MB
Func<int> getLength = () => huge.Length;
RegisterCallback(getLength);            // stored long-term
// huge is "done" but...
```
<details><summary>Answer</summary>

The closure **captures `huge`** (the whole 100MB array), keeping it alive as long as `getLength` is referenced → effectively a leak. **Fix:** capture only what you need — `int len = huge.Length; Func<int> f = () => len;` — so the array can be collected. Closures can silently root large objects.
</details>
