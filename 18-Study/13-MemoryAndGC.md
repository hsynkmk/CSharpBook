# 13 — Memory & Garbage Collection ⭐ (high-frequency)

## ⚡ 30-second answer

.NET is **garbage-collected**: the GC automatically reclaims heap objects no longer **reachable** from roots (stack, statics, registers). It's a **generational, tracing, compacting** collector — most objects die young, so it splits the heap into **Gen 0/1/2** and collects Gen 0 frequently/cheaply, promoting survivors. Large objects (≥85KB) go on the **Large Object Heap (LOH)**. You don't free memory, but you **must release unmanaged/external resources** via **`IDisposable`** (`using`). A "memory leak" in .NET = **unintended reachability** — something long-lived still references objects you expect gone (events, statics, caches, closures).

---

## Core mechanics

**Generations** (the key model):
- **Gen 0** — newly allocated; collected often, very fast (most objects die here).
- **Gen 1** — survived one GC; a buffer between 0 and 2.
- **Gen 2** — long-lived; collected rarely (full/expensive collection).
- Survivors are **promoted** up; the GC **compacts** (moves live objects together, eliminating fragmentation) and updates references.

**LOH** — objects ≥85KB (big arrays/strings). Collected with Gen 2, **not compacted by default** → can fragment. Mitigate with `ArrayPool<T>` / pooling.

**IDisposable** — deterministic cleanup of unmanaged resources (file handles, sockets, DB connections):
```csharp
using var conn = new SqlConnection(cs);   // Dispose() called at end of scope
// full pattern for a class owning unmanaged resources:
public void Dispose() { Dispose(true); GC.SuppressFinalize(this); }
protected virtual void Dispose(bool disposing) {
    if (_disposed) return;
    if (disposing) { _managed?.Dispose(); }   // dispose owned managed resources
    _handle?.Dispose();                         // release unmanaged (SafeHandle preferred)
    _disposed = true;
}
```

**Finalizers (`~Type()`)** — last-resort cleanup if `Dispose` wasn't called; run on a separate finalizer thread, **non-deterministic**, and **delay collection** (object survives an extra GC). Prefer **`SafeHandle`** over writing finalizers.

**`Span<T>`/`Memory<T>`** — views over contiguous memory with **no allocation/copy**; `stackalloc` + `Span` for stack buffers; `ArrayPool<T>` to reuse large arrays.

---

## Comparison tables

| Generation | Collected | Cost | Holds |
|---|---|---|---|
| Gen 0 | very often | cheap | new objects |
| Gen 1 | sometimes | moderate | short survivors |
| Gen 2 | rarely | expensive (full) | long-lived |
| LOH | with Gen 2 | expensive, not compacted | objects ≥85KB |

| | `IDisposable` / `using` | Finalizer (`~`) |
|---|---|---|
| When runs | deterministically (end of scope) | non-deterministically (GC) |
| Thread | caller | finalizer thread |
| Cost | none extra | delays collection by a GC cycle |
| Use | always for owned resources | last-resort safety net (prefer SafeHandle) |

---

## 🪤 Traps & gotchas (memory leaks in a GC'd runtime)

The classic leak = **unintended reachability**:
- **Event handlers**: a publisher holds the subscriber via the delegate. Short-lived subscriber + long-lived publisher + no `-=` → never collected ([03](03-Generics-Delegates-Events.md)). Unsubscribe in `Dispose` / use weak events.
- **Static fields / static caches**: anything reachable from a `static` lives forever. Unbounded static caches grow without limit — bound/evict them.
- **Captured closures**: a long-lived delegate capturing a big object keeps it alive.
- **Timers / background tasks**: a running `Timer`/`Task` referencing your object keeps it rooted; stop/dispose them.
- **Not disposing** `IDisposable` → leaked OS handles (sockets, files) even though managed memory is fine; `HttpClient` socket exhaustion if created per request ([18](18-Caching-Resilience-Http.md)).
- **LOH fragmentation**: frequent large-array churn fragments the LOH → growing memory. Pool buffers (`ArrayPool<T>`).
- **Calling `GC.Collect()` manually**: almost always wrong — interferes with the GC's heuristics.
- **Forgetting `GC.SuppressFinalize(this)`** in `Dispose` makes a finalizable object survive an extra GC.

---

## ❓ Likely questions

**Q: How does the GC decide what to collect?**
A: It traces from roots (stack, statics, registers); anything **unreachable** is garbage. It's generational — collects Gen 0 often, promotes survivors, rarely does a full Gen 2 collection.

**Q: Why generations?**
A: The generational hypothesis — most objects die young. Collecting only Gen 0 (small, frequent, cheap) and rarely touching long-lived Gen 2 makes GC efficient.

**Q: What is the LOH and why care?**
A: Large Object Heap for objects ≥85KB; collected with Gen 2 and not compacted by default → fragmentation. Pool/reuse large buffers (`ArrayPool<T>`).

**Q: If .NET has a GC, can you still leak memory?**
A: Yes — "leak" means unintended reachability: events, static caches, closures, or timers keep objects rooted so the GC can't reclaim them.

**Q: What's the `IDisposable` pattern for?**
A: Deterministic release of unmanaged/external resources (handles, connections). Use `using`; implement the Dispose pattern (+ `SuppressFinalize`) when a type owns such resources.

**Q: When do you write a finalizer?**
A: Rarely — only as a safety net for unmanaged resources if `Dispose` wasn't called. Prefer `SafeHandle`. Finalizers delay collection and run non-deterministically.

**Q: `Span<T>` vs `Memory<T>`?**
A: `Span<T>` is stack-only (a ref struct) — fastest, can't be a field/async local. `Memory<T>` is heap-storable (fields, across `await`) and yields a `Span` via `.Span`. Span for synchronous hot paths, Memory when you need to persist/await.

**Q: Should you call `GC.Collect()`?**
A: No, except niche cases. The GC self-tunes; manual collection usually hurts.

---

## 🎓 Senior Extra

- **Workstation vs Server GC**: Workstation (low latency, one heap, default for client) vs **Server GC** (a heap + dedicated GC thread per core, higher throughput, default for ASP.NET Core on multi-core). **DATAS** (.NET 8+) dynamically adapts Server GC heap count to load to reduce memory.
- **Background/concurrent GC** collects Gen 2 mostly concurrently with the app to minimize pauses; **write barriers** track cross-generation references (the card table) so Gen 0 collection doesn't scan Gen 2.
- **GC modes & latency**: `GCLatencyMode` (e.g., `SustainedLowLatency`) and `GC.TryStartNoGCRegion` for latency-critical sections — advanced.
- **Allocation is the lever**: GC cost scales with allocation rate, not heap size per se. Reducing allocations (Span, pooling, structs, avoiding LINQ/closures in hot loops) is the main perf win ([21](21-Deployment-Perf-Tooling.md)).
- **Finding leaks**: `dotnet-dump` → `dumpheap -stat` (what's eating the heap) → `gcroot <addr>` (the retention path to the root) — the canonical leak hunt ([21](21-Deployment-Perf-Tooling.md)).
- **`SafeHandle`** wraps OS handles with built-in finalization + ref-counting (resists handle-recycling races) — the right way to own unmanaged handles instead of a hand-written finalizer.
- **Pinning** (`fixed`/`GCHandle.Pinned`) prevents the GC from moving an object (for interop) but harms compaction — keep pins short.
- **`IAsyncDisposable`** for async cleanup (flush/close over the wire) — `await using` ([10](10-AsyncAwait.md)).
- **Escape analysis (.NET 10)**: the JIT may stack-allocate provably non-escaping objects, cutting heap pressure automatically.

→ Deeper: [`../CSharpBook/09-MemoryPerformance/`](../CSharpBook/09-MemoryPerformance/README.md), [`../DotNetBook/01-Runtime/04-GCDeepDive.md`](../DotNetBook/01-Runtime/README.md)
