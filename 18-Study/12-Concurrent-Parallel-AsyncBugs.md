# 12 — Concurrent Collections, Parallelism & Classic Async Bugs ⭐⭐

## ⚡ 30-second answer

For shared mutable data across threads, use **concurrent collections** (`ConcurrentDictionary`, `ConcurrentQueue`, `BlockingCollection`) or **`Channel<T>`** (the modern async producer/consumer pipeline) instead of locking a plain collection. For CPU-bound data-parallel work use the **TPL** (`Parallel.For`/`ForEach`, `Parallel.ForEachAsync`, PLINQ). And know the **classic async bugs** cold — they're the most-asked traps: **sync-over-async (deadlock + starvation)**, **`async void`**, **fire-and-forget**, **thread-pool starvation**.

---

## Core mechanics

**Concurrent collections** (thread-safe, no external lock):
```csharp
var cache = new ConcurrentDictionary<string,int>();
cache.AddOrUpdate(key, 1, (_, old) => old + 1);     // atomic per key
cache.GetOrAdd(key, k => Load(k));                   // factory may run >1x under race (only one kept)

var q = new ConcurrentQueue<Work>();  q.Enqueue(w);  q.TryDequeue(out var item);
```

**`Channel<T>`** — async producer/consumer with backpressure:
```csharp
var ch = Channel.CreateBounded<int>(100);            // bounded → backpressure
await ch.Writer.WriteAsync(item, ct);                // waits if full
await foreach (var item in ch.Reader.ReadAllAsync(ct)) { Process(item); }
```

**Parallelism (CPU-bound)**:
```csharp
Parallel.For(0, n, i => Compute(i));                          // data parallelism across cores
await Parallel.ForEachAsync(items,
    new ParallelOptions { MaxDegreeOfParallelism = 20 },      // bounded concurrency
    async (item, ct) => await CallApiAsync(item, ct));        // async + throttled
var results = source.AsParallel().Where(...).Select(...);     // PLINQ
```

**`BlockingCollection<T>`** — older bounded producer/consumer (blocking, not async) — prefer `Channel<T>` for async.

---

## Comparison tables

| Need | Use |
|---|---|
| Thread-safe key/value | `ConcurrentDictionary` |
| Thread-safe FIFO/LIFO | `ConcurrentQueue` / `ConcurrentStack` |
| Async producer/consumer pipeline | **`Channel<T>`** |
| Bounded blocking producer/consumer | `BlockingCollection<T>` |
| CPU-bound loop over data | `Parallel.For/ForEach`, PLINQ |
| Async over a collection, throttled | **`Parallel.ForEachAsync`** |

| Parallel vs Concurrent vs Async | Means |
|---|---|
| **Parallel** | doing many things at once on multiple cores (CPU-bound) |
| **Concurrent** | managing many things in progress (may interleave on few threads) |
| **Async** | not blocking while waiting (I/O-bound; frees the thread) |

---

## 🪤 The classic async bugs (memorize these)

1. **Sync-over-async** — `task.Result` / `task.Wait()` blocking on async work.
   - **Deadlock** in a sync-context (UI/legacy ASP.NET): the blocked thread is the one the continuation needs.
   - **Thread-pool starvation** in ASP.NET Core: each blocked request holds a pool thread → under load the pool can't keep up → latency cliff.
   - **Fix**: `await` all the way up; never `.Result`/`.Wait()`.

2. **`async void`** — exceptions can't be caught and **crash the process**; can't be awaited. Only for event handlers. **Fix**: `async Task`.

3. **Fire-and-forget** — not awaiting a task: exceptions swallowed (unobserved), ordering/lifecycle breaks, it may run after the scope is gone. **Fix**: await it, or deliberately manage background work (a queue / `BackgroundService` — [20](20-Observability-Messaging-Background.md)) and observe exceptions.

4. **Thread-pool starvation** — blocking pool threads (sync I/O, `.Result`, long locks) faster than the pool injects threads. Symptoms: queue length + thread count climbing in `dotnet-counters` ([21](21-Deployment-Perf-Tooling.md)). **Fix**: async I/O; don't block.

5. **Over-parallelizing I/O** — `Task.WhenAll` over 10,000 HTTP calls floods the server/sockets. **Fix**: bound concurrency with `Parallel.ForEachAsync` (MaxDegreeOfParallelism) or a `SemaphoreSlim`.

6. **Non-thread-safe collection shared** — writing a plain `Dictionary`/`List` from multiple threads corrupts it. **Fix**: concurrent collection or lock.

7. **`GetOrAdd` factory side effects** — the value factory can run more than once under contention. **Fix**: don't rely on it running exactly once; use `Lazy<T>` as the value if the factory must be single-shot.

---

## ❓ Likely questions

**Q: How is `ConcurrentDictionary` thread-safe?**
A: Lock-free reads and lock-striped writes (multiple locks guard buckets), so different keys can be updated concurrently. `AddOrUpdate`/`GetOrAdd` are atomic per key (but the value factory may run more than once under races).

**Q: `Channel<T>` vs `BlockingCollection<T>`?**
A: `Channel<T>` is async-first (`WriteAsync`/`ReadAllAsync`) with bounded backpressure — ideal for async pipelines. `BlockingCollection` blocks threads. Prefer Channels for modern async producer/consumer.

**Q: When use `Parallel.For` vs `Task.WhenAll` vs `Parallel.ForEachAsync`?**
A: `Parallel.For` for CPU-bound loops across cores. `Task.WhenAll` to await many already-async operations. `Parallel.ForEachAsync` for async work over a collection with a concurrency cap.

**Q: Difference between concurrency and parallelism?**
A: Concurrency = dealing with many things in progress (interleaving). Parallelism = doing many things simultaneously on multiple cores. Async enables concurrency for I/O without parallelism.

**Q: Why does `Task.Run(() => Foo()).Result` risk deadlock/starvation?**
A: It blocks the calling thread on async work; in a sync context that's a deadlock, and in ASP.NET Core it ties up pool threads → starvation. Await instead.

**Q: How do you limit concurrency of async calls?**
A: `Parallel.ForEachAsync` with `MaxDegreeOfParallelism`, or a `SemaphoreSlim(n)` you `WaitAsync` before each call.

**Q: What is fire-and-forget and why is it dangerous?**
A: Starting a task without awaiting it — exceptions go unobserved, completion/ordering isn't guaranteed, and it may outlive its context. Await or route to managed background processing.

---

## 🎓 Senior Extra

- **`ConcurrentDictionary` internals**: lock striping for writes, volatile reads for lock-free gets. Sizing concurrency level and capacity reduces contention. Iterating it is a moving snapshot (weakly consistent).
- **Backpressure** is the point of bounded `Channel<T>`: a slow consumer makes producers `await` on `WriteAsync`, preventing unbounded memory growth — critical for ingestion pipelines ([20](20-Observability-Messaging-Background.md)).
- **PLINQ caveats**: overhead + ordering costs; only wins for CPU-bound work over large sequences; `AsOrdered()` preserves order at a cost; not for I/O.
- **`Parallel.ForEachAsync`** (.NET 6) is the idiomatic bounded async fan-out; before it, people hand-rolled `SemaphoreSlim` + `Task.WhenAll` (still valid).
- **Detecting the bugs in prod**: `dotnet-counters` shows thread-pool **queue length** & **thread count** (starvation) and **lock contention**; a `dotnet-dump` shows threads stuck on `.Result`/`Monitor.Wait` (deadlock) ([21](21-Deployment-Perf-Tooling.md)).
- **`TaskScheduler.UnobservedTaskException`** fires for unobserved faulted tasks (fire-and-forget leaks) — wire it up in testing.
- **Async + cancellation + timeouts** compose with linked `CancellationTokenSource`; combine a request token with `CancelAfter` for per-operation deadlines ([10](10-AsyncAwait.md), [18](18-Caching-Resilience-Http.md)).
- **Lock-free isn't always faster**: under low contention a plain `lock` can beat a CAS-retry loop; measure.

→ Deeper: [`../CSharpBook/08-Concurrency/`](../CSharpBook/08-Concurrency/README.md)
