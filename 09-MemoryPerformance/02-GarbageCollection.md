# Garbage Collection

## What it is

The **GC** automatically reclaims heap memory that's no longer reachable from your program. .NET's GC is **generational**, **tracing**, and (mostly) **stop-the-world**. Understanding it is foundational to writing high-performance .NET code.

```csharp
var data = new byte[1_000_000];   // 1 MB on the heap
// ... use data ...
data = null;                        // reference gone

// Eventually, the GC notices the array is unreachable and reclaims it.
```

You never call `delete` — the GC tracks reachability. The trade-off: occasional **collection pauses** when the GC scans the heap.

.NET 10 makes **DATAS** (Dynamically Adapting To Application Sizes) the default GC mode — it auto-tunes itself per workload. Major win for cloud / container scenarios.

---

## Why it exists

Manual memory management (C, C++) has well-known problems:
- **Use-after-free** — accessing memory after `delete`.
- **Double-free** — freeing the same memory twice.
- **Memory leaks** — forgetting to free.
- **Buffer overruns** — writing past allocated memory.

Automatic GC eliminates these for managed memory. You pay a small runtime cost; you get safety and (mostly) freedom from worrying about lifetimes.

---

## Generations

The CLR splits the heap into **generations**:

| Generation | Holds | Collection frequency | Cost |
|---|---|---|---|
| **Gen0** | Newly allocated objects | Very often | Cheap (~ms) |
| **Gen1** | Survived one Gen0 collection | Sometimes | Medium |
| **Gen2** | Long-lived objects | Rarely | Expensive (~tens of ms) |
| **LOH** (Large Object Heap) | Objects ≥85,000 bytes | Only in Gen2 | Expensive, not compacted |
| **POH** (Pinned Object Heap) | Pinned objects (since .NET 5) | Like LOH | Avoids fragmenting Gen2 |

The **generational hypothesis**: most objects die young. Collecting Gen0 frequently is cheap and recovers most garbage; Gen2 collections are rare but expensive.

### How an object moves between generations

```
New object → allocated in Gen0
        → survives a Gen0 GC → promoted to Gen1
        → survives a Gen1 GC → promoted to Gen2
        → in Gen2 forever (or until unreachable + Gen2 GC)
```

Each generation has a budget. When the budget is hit, a GC happens — collecting that generation and all younger ones.

---

## What happens during a GC

1. **Pause** all managed threads (stop-the-world).
2. **Mark**: walk from "roots" (static fields, thread stacks, GC handles), follow all references, mark reachable objects.
3. **Compact** (typically): unreachable objects are gaps; compact the heap to remove gaps; update all references.
4. **Resume** threads.

For Gen0: only scan Gen0 (plus refs **into** Gen0 from older generations via **card tables**).
For Gen2 (full GC): scan everything.

### Stop-the-world ≠ "blocks for seconds"

For typical apps:
- Gen0 GC: ~1-10 ms.
- Gen1 GC: ~5-50 ms.
- Gen2 GC: ~10-200 ms.

For high-end servers with multi-GB heaps: Gen2 can take hundreds of ms or worse without proper tuning.

**Background GC** (default since .NET 4.5) runs Gen2 concurrently with application threads — significantly reduces pauses.

---

## GC modes

### Workstation GC
Default for desktop apps. Single-threaded GC, optimized for low latency on a single user-facing app.

### Server GC
Multi-threaded GC. Each CPU has its own heap and dedicated GC thread. Optimized for **throughput** at the cost of higher memory.

```xml
<!-- in csproj or runtimeconfig.json -->
<PropertyGroup>
    <ServerGarbageCollection>true</ServerGarbageCollection>
</PropertyGroup>
```

For ASP.NET Core servers: Server GC is the default.

### Concurrent (Background) GC
Gen2 GC happens concurrently with app threads. Default on. Trades a bit of throughput for much lower pauses.

```xml
<ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
```

### DATAS (Dynamically Adapting To Application Sizes) — .NET 8+, default in .NET 10

DATAS adjusts heap size and GC behavior **based on the running app's allocation pattern**.

Before DATAS: Server GC pre-allocated huge heaps based on CPU count (one heap per core). For a small container with 8 cores allocated but only using 100 MB, you'd waste GB of address space.

DATAS: starts small, grows as needed, shrinks when idle. Critical for cloud / container deployments.

Enable / disable:
```xml
<GarbageCollectionAdaptationMode>1</GarbageCollectionAdaptationMode>   <!-- 1 = on, 0 = off -->
```

In .NET 10, it's **on by default**. Most apps see better memory efficiency without performance regressions.

---

## Triggering a GC

Almost never do this manually:

```csharp
GC.Collect();                                  // force a full Gen2 GC
GC.Collect(2, GCCollectionMode.Forced, true);   // exhaustive
GC.WaitForPendingFinalizers();
```

When it's OK:
- Diagnosing memory issues (force a collection, then measure).
- After a known one-time burst of garbage (e.g., parsing a 1 GB file, then settling into low-allocation mode).

In production code: trust the GC. Manually-triggered collections often hurt more than help.

---

## Roots — what keeps objects alive

An object is **reachable** (alive) if you can trace a chain of references from a **root** to it. Roots include:

- **Static fields** of any loaded type.
- **Thread stacks** — local variables, parameters in active method calls.
- **GC handles** — manually pinned objects, weak references with the handle still alive.
- **CPU registers** — temporary references being processed.
- **Finalizer queue** — objects waiting to be finalized.

If no chain leads to an object, it's garbage; the GC reclaims it.

**To NOT keep an object alive**: clear all references. If a static collection holds it, remove it from the collection. If an event handler captures it, unsubscribe.

---

## Common GC stats (dotnet-counters)

```
$ dotnet-counters monitor -p <pid> --counters System.Runtime
```

Look at:
- **`gc-heap-size`** — total bytes allocated across all generations.
- **`gen-0-gc-count` / `gen-1-gc-count` / `gen-2-gc-count`** — per-generation counts.
- **`gen-0-size` / `gen-1-size` / `gen-2-size` / `loh-size` / `poh-size`** — per-generation sizes.
- **`% time in gc`** — how much time has been spent in GC vs application.
- **`alloc-rate`** — bytes allocated per second.
- **`gc-fragmentation`** — heap fragmentation %.

For healthy production:
- % time in GC < 5%.
- Gen2 collections rare (< 1 per minute typically).
- Alloc rate stable (not climbing over time).

If Gen2 size keeps growing → **memory leak**. See [§13 MemoryLeaks](13-MemoryLeaks.md).

---

## Allocation patterns and what they cost

```csharp
// Each call: 24-byte allocation (Object header + 8-byte string)
for (int i = 0; i < 1_000_000; i++) {
    string s = i.ToString();
}
// Total: ~24 MB of Gen0 garbage. Triggers several Gen0 collections.
```

```csharp
// LOH allocation — ≥85,000 bytes
var bigArray = new byte[100_000];
// Goes to LOH. Not compacted by default. Stays until Gen2 GC.
```

```csharp
// Pinned — POH (since .NET 5)
GCHandle.Alloc(obj, GCHandleType.Pinned);
// Pinned objects fragment normal heap; POH isolates them.
```

```csharp
// Boxed in a loop — Gen0 churn
for (int i = 0; i < 1_000_000; i++) {
    object o = i;   // boxing
}
// ~24 MB of garbage.
```

For high-allocation workloads:
- **Reuse buffers**: `ArrayPool<T>.Shared.Rent()`.
- **Avoid boxing**: generic constraints, not interface params.
- **Strings**: `StringBuilder`, `Span<char>`, `string.Create`, interpolated handlers.
- **Use structs** for small short-lived values.

---

## LOH — the Large Object Heap

Objects ≥ **85,000 bytes** (~83 KB) go to the LOH. Why the threshold? Originally, the cost of copying a large object during compaction outweighed the cost of leaving gaps.

Properties:
- Not compacted by default (configurable: `GCHeapCompactionMode.CompactOnce`).
- Collected only during Gen2 GC.
- Fragmentation possible if you allocate/free large objects of varying sizes.

For large buffers, use **`ArrayPool<byte>.Shared.Rent(size)`** instead of `new byte[size]` — the pool reuses arrays, avoiding LOH churn.

Modern API: `GC.AllocateUninitializedArray<T>(size)` skips zeroing for huge arrays — useful when you'll immediately fill it.

---

## POH — the Pinned Object Heap

Pinned objects can't be moved by the GC (because something native is holding a pointer to them). Pre-.NET 5, pinned objects fragmented Gen2.

POH (since .NET 5): a separate heap for pinned allocations. Allocate explicitly:

```csharp
byte[] pinnedBuffer = GC.AllocateArray<byte>(1024, pinned: true);
```

For long-lived pinning (P/Invoke buffers held across calls), POH avoids fragmenting your main heap.

---

## Finalizers and the GC

Objects with a finalizer (`~ClassName`) are **not collected immediately** when unreachable. Instead:

1. GC sees the object is unreachable.
2. Marks it for finalization, moves it to the **finalizer queue**.
3. The dedicated finalizer thread later runs the finalizer.
4. Object is now eligible for collection; **next** GC reclaims it.

Net effect: finalizers cause objects to survive **two** GC cycles. Plus the finalizer thread runs sequentially — slow finalizers block all finalization.

**Avoid finalizers** for general code. Use them only as a safety net for unmanaged resources (after IDisposable). Modern alternative: **SafeHandle** (which has its own optimized finalization). See [§04 Finalizers](04-Finalizers.md).

---

## Internals — how marking works

When a GC runs:

1. **Suspend all managed threads** — the runtime walks each thread's stack to identify references.
2. **Roots**: scan static fields, stack frames, registers, GC handles. Push all referenced objects to a work queue.
3. **Mark phase**: pop an object; mark it; push everything it references. Repeat until queue empty.
4. **Plan phase**: identify gaps (unreachable objects).
5. **Compact phase** (most generations): shift live objects to fill gaps; update pointers.
6. **Sweep phase** (LOH): just mark gaps as free; don't move.
7. **Resume** application threads.

For Gen0/Gen1 collections, only the youngest generations are walked. Cross-generation references (from Gen2 → Gen0) are tracked via **card tables** — a bitmap that flags regions of Gen2 where references to younger gen may exist. The GC scans those regions during Gen0 collection.

Card tables make Gen0 collections fast: you don't have to scan all of Gen2 to find Gen0 references; just the marked card pages.

---

## Background GC

Server background GC runs Gen2 work concurrently with application threads:

1. App threads keep running.
2. GC marks live objects in Gen2 concurrently.
3. **Brief pause** for ephemeral (Gen0/1) collection.
4. App threads resume.
5. GC sweeps / compacts (mostly concurrently).

Result: Gen2 pauses go from 100+ ms to ~10 ms in many workloads. .NET 10 has further reduced even those pauses.

---

## Write barriers

When you assign a reference to a field of an older-generation object:

```csharp
gen2Object.Field = newGen0Object;
```

The runtime emits a **write barrier** — code that updates the card table for the region containing `gen2Object`. This marks "there's a young reference here," so the next Gen0 GC knows to scan this card.

Write barriers add a small cost (~1-2 ns) to every reference field write. .NET 10 reduces this cost — some assignments can skip the barrier.

For high-frequency write-heavy workloads, this can be measurable. Generally not a concern for normal code.

---

## .NET 10 improvements

The .NET 10 GC has several wins over .NET 8:

- **DATAS as default**: adaptive heap sizing.
- **Reduced write barrier cost**: ~30% faster in microbenchmarks for some patterns.
- **Better Gen2 throughput**: improved partitioning, less contention.
- **NUMA-aware** allocation on multi-socket machines.
- **Bulk free for LOH** — faster reclaim of large arrays.

For most apps, upgrading to .NET 10 yields better memory behavior with no code changes.

---

## Common bugs

### Memory leak via static cache

```csharp
public static class Cache {
    private static readonly Dictionary<string, byte[]> _items = new();
    public static void Add(string key, byte[] item) => _items[key] = item;
}

// Without eviction, _items grows forever → memory leak via static root
```

Static fields are GC roots. Never freed (until process exits). Use bounded caches (`MemoryCache`) or weak references.

### Event handler holding subscriber

```csharp
publisher.OnUpdated += subscriber.HandleUpdate;
// subscriber's instance is now kept alive as long as publisher is alive
```

Forgetting to unsubscribe creates a leak. Use weak event patterns or explicit unsubscribe in IDisposable.

### Boxing in tight loops

```csharp
for (int i = 0; i < 1_000_000; i++) {
    list.Add(i);   // if list is ArrayList (non-generic), each Add boxes — 1M allocations
}
```

Use `List<int>` (generic). Avoids boxing entirely.

### Async state machine boxes

```csharp
async Task M() {
    await Task.Delay(100);
}
```

On the first suspension, the state machine struct is boxed. For methods called millions of times, this allocates a lot. .NET 8+ has box-elision optimizations; .NET 10 improves further. For ultra-hot paths, prefer `ValueTask` with sync-completion fast path.

### Holding LOH arrays for too long

```csharp
private byte[]? _bigBuffer;
public void Use() {
    _bigBuffer = new byte[100_000];  // LOH, ~85 KB
    Process(_bigBuffer);
}
// _bigBuffer is now a long-lived LOH allocation.
```

Either reuse (ArrayPool) or null it out after use.

---

## Performance summary

| Operation | Approx cost |
|---|---|
| `new SmallClass()` | ~10-20 ns |
| Gen0 GC | ~ms (depends on heap size) |
| Gen2 GC (full) | ~tens of ms (depends on heap size) |
| Background Gen2 GC | mostly invisible to app threads |
| Write barrier per field assignment | ~1-2 ns |
| `Span<T>` slice (no alloc) | ~free |
| `stackalloc N` | ~free + stack space |
| `ArrayPool<T>.Rent` | ~50-100 ns (vs ~10 ns for new) but avoids LOH thrashing |

The big wins are usually **eliminating allocations from hot paths**, not optimizing individual allocations.

---

## When to think about GC

You should:
- When `% time in gc` > 10% in production.
- When latency-sensitive code has periodic spikes.
- When designing libraries used in tight loops.
- When implementing high-throughput services (parsers, queues, web servers at scale).

You should NOT:
- For typical CRUD apps.
- During premature optimization.
- Before profiling.

---

## Tools

- **dotnet-counters** — live counters.
- **dotnet-trace** — sample allocations.
- **dotnet-gcdump** — heap snapshots.
- **dotnet-dump + sos** — analyze dumps.
- **PerfView** — comprehensive Windows profiler (still gold standard).
- **JetBrains dotMemory** — UI-driven; great for finding leaks.

[DotNetBook Chapter 19 — Performance & Tooling](../../DotNetBook/19-Performance/README.md) goes deep on these.

---

## Summary

- .NET's GC is generational, tracing, with Gen0 / Gen1 / Gen2 / LOH / POH.
- Most objects die young — Gen0 collections are cheap, frequent.
- Gen2 collections are rare, expensive — but background GC overlaps with app threads.
- DATAS (.NET 8+, default in .NET 10) auto-tunes heap size for the workload.
- Static fields are GC roots — they keep their referenced objects alive.
- For perf: minimize allocations on hot paths; use Span, ArrayPool, structs.

→ Next: [03-IDisposable.md](03-IDisposable.md)
