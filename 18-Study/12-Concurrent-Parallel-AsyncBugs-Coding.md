# 12 — Concurrent, Parallel & Async Bugs — Coding Questions ⭐⭐

> Find the bug / predict the output. The async-bug questions here are the most-asked traps. (Concepts: [12-Concurrent-Parallel-AsyncBugs.md](12-Concurrent-Parallel-AsyncBugs.md))

---

### Q1 — async inside Parallel.ForEach (very common bug)
```csharp
Parallel.ForEach(urls, async url => {
    var data = await httpClient.GetStringAsync(url);
    Process(data);
});
Console.WriteLine("done");   // ?
```
<details><summary>Answer</summary>

**Broken.** `Parallel.ForEach` doesn't understand async — the lambda is `async void`, so `ForEach` returns **before** any download finishes, exceptions are lost, and "done" prints immediately while work is still pending (or crashes). **Fix:** `await Parallel.ForEachAsync(urls, async (url, ct) => { ... });`
</details>

---

### Q2 — Non-thread-safe collection
```csharp
var results = new List<int>();
Parallel.For(0, 1000, i => results.Add(i));
Console.WriteLine(results.Count);   // ?
```
<details><summary>Answer</summary>

**Less than 1000, or throws/corrupts** — `List<T>.Add` isn't thread-safe; concurrent adds race on the internal array/index. **Fix:** `ConcurrentBag<int>`/`ConcurrentQueue<int>`, or lock, or use `results` per-partition then merge.
</details>

---

### Q3 — Channel producer/consumer
```csharp
var ch = Channel.CreateUnbounded<int>();
var producer = Task.Run(async () => {
    for (int i = 0; i < 3; i++) await ch.Writer.WriteAsync(i);
    ch.Writer.Complete();
});
await foreach (var x in ch.Reader.ReadAllAsync()) Console.Write(x);
await producer;
```
<details><summary>Answer</summary>

**`012`**. The consumer reads as items arrive; `Complete()` ends the `await foreach` loop cleanly. Forgetting `Complete()` would make `ReadAllAsync` hang forever waiting for more items.
</details>

---

### Q4 — Bounded channel = backpressure
```csharp
var ch = Channel.CreateBounded<int>(2);   // capacity 2
// producer with no consumer running:
await ch.Writer.WriteAsync(1);
await ch.Writer.WriteAsync(2);
await ch.Writer.WriteAsync(3);   // ?
```
<details><summary>Answer</summary>

The third `WriteAsync` **blocks (awaits)** until a consumer reads — the channel is full (capacity 2). That's **backpressure**: producers slow to match consumers, preventing unbounded memory growth. (Unbounded channels never block writers but risk OOM.)
</details>

---

### Q5 — Closure capture in Parallel.For
```csharp
var tasks = new List<Task>();
for (int i = 0; i < 3; i++)
    tasks.Add(Task.Run(() => Console.Write(i)));
await Task.WhenAll(tasks);
```
<details><summary>Answer</summary>

Likely **`333`** (or some mix) — the `for` loop variable `i` is captured by reference; by the time tasks run, `i` is 3. **Fix:** `int copy = i; tasks.Add(Task.Run(() => Console.Write(copy)));`. Same closure trap as [03](03-Generics-Delegates-Events-Coding.md), now with threads making it non-deterministic.
</details>

---

### Q6 — Sync-over-async starvation reproduction
```csharp
// thread pool min = few; each call blocks:
var tasks = Enumerable.Range(0, 100).Select(_ => Task.Run(() => SlowAsync().Result));
await Task.WhenAll(tasks);
```
<details><summary>Answer</summary>

**Thread-pool starvation / very slow.** Each `Task.Run` takes a pool thread and **blocks** it on `.Result`; with limited threads injected slowly, the 100 tasks serialize/stall. **Fix:** don't block — `await SlowAsync()` directly (no `Task.Run`, no `.Result`).
</details>

---

### Q7 — ConcurrentDictionary factory runs twice
```csharp
var cache = new ConcurrentDictionary<int,Guid>();
Parallel.For(0, 2, _ => cache.GetOrAdd(1, k => { Console.WriteLine("create"); return Guid.NewGuid(); }));
```
<details><summary>Answer</summary>

"create" may print **twice** under contention — the value factory isn't guaranteed single-invocation (only one result is kept). Don't rely on it for one-time side effects. For single-shot, store a `Lazy<Guid>` as the value: `cache.GetOrAdd(1, _ => new Lazy<Guid>(Guid.NewGuid)).Value`.
</details>

---

### Q8 — PLINQ ordering
```csharp
var result = Enumerable.Range(1, 10).AsParallel().Select(x => x * 10);
Console.WriteLine(string.Join(",", result));
```
<details><summary>Answer</summary>

Order is **not guaranteed** (parallel partitions complete out of order). Add **`.AsOrdered()`** to preserve input order (at a cost), or use `.OrderBy` after. Also: PLINQ only helps **CPU-bound** work over large sequences — not I/O.
</details>

---

### Q9 — Limiting concurrency with SemaphoreSlim
```csharp
var sem = new SemaphoreSlim(3);   // max 3 at once
var tasks = urls.Select(async url => {
    await sem.WaitAsync();
    try { return await httpClient.GetStringAsync(url); }
    finally { sem.Release(); }
});
var all = await Task.WhenAll(tasks);
```
<details><summary>Answer</summary>

**Correct** bounded fan-out — all tasks start, but only **3 run the HTTP call concurrently** (the rest await the semaphore). Equivalent to `Parallel.ForEachAsync(..., MaxDegreeOfParallelism=3)`. `Release()` in `finally` is essential.
</details>

---

### Q10 — Spot all the bugs (senior)
```csharp
public async void ProcessOrders(List<Order> orders) {
    var results = new List<Result>();
    Parallel.ForEach(orders, async o => {
        var r = await _api.SubmitAsync(o).Result;   // ...
        results.Add(r);
    });
}
```
<details><summary>Answer</summary>

**Four bugs:** (1) `async void` method → uncatchable exceptions; (2) `Parallel.ForEach` with an `async` lambda → fire-and-forget, returns before work done; (3) `await ....Result` → both awaiting *and* blocking (redundant + deadlock/starvation risk); (4) `results.Add` from multiple threads → race on a non-thread-safe `List`. **Fix:** `public async Task ProcessOrders(...)` + `var results = await Task.WhenAll(orders.Select(o => _api.SubmitAsync(o)));` (bounded if many).
</details>
