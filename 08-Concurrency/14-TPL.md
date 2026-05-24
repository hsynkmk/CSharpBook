# TPL — Parallel, PLINQ, and Friends

## What it is

The **Task Parallel Library** (TPL) is .NET's high-level data-parallelism API. While `async/await` is for I/O concurrency, TPL is for **CPU-bound parallelism** — splitting work across cores.

```csharp
Parallel.For(0, 1_000_000, i => HeavyCompute(i));         // CPU-bound parallel loop
Parallel.ForEach(items, item => Process(item));            // parallel collection iteration

await Parallel.ForEachAsync(urls, async (url, ct) => {     // async-aware parallel
    await DownloadAsync(url, ct);
});

// PLINQ
var result = items.AsParallel().Where(x => Heavy(x)).Sum();
```

The TPL was added in .NET 4.0 (2010), before async/await. It's still relevant for data-parallel workloads.

---

## Parallel.For — index-based parallel loop

```csharp
Parallel.For(0, 1_000_000, i => {
    array[i] = Compute(i);
});
```

The .NET runtime partitions the index range across cores and runs the body in parallel. Each iteration runs the body once, on whatever thread is free.

### With state aggregation (thread-local)

```csharp
long total = 0;
Parallel.For(0, 1_000_000,
    () => 0L,                                // initial local state per thread
    (i, _, localSum) => localSum + i * i,    // body, returns updated local
    finalLocal => Interlocked.Add(ref total, finalLocal)   // combine on completion
);
```

Each thread accumulates its own `localSum`. At the end, all locals are combined via the final callback. Far faster than a shared `Interlocked.Add` per iteration (one combine per thread vs one atomic per iteration).

### Options

```csharp
Parallel.For(0, n, new ParallelOptions {
    MaxDegreeOfParallelism = 4,           // limit concurrent threads
    CancellationToken = ct,
    TaskScheduler = TaskScheduler.Default
}, body);
```

`MaxDegreeOfParallelism` is the main control. By default, uses thread pool — usually all logical cores.

---

## Parallel.ForEach — collection iteration

```csharp
Parallel.ForEach(items, item => Process(item));
```

Same shape, but iterates an `IEnumerable<T>`. Partitions the source into chunks, distributes to workers.

With state aggregation:
```csharp
double total = 0;
Parallel.ForEach(items,
    () => 0.0,
    (item, _, localSum) => localSum + item.Value,
    finalLocal => Interlocked.Exchange(ref total, total + finalLocal)
);
```

Used for "sum/count/aggregate across a collection" patterns.

---

## Parallel.ForEachAsync — async-aware (.NET 6+)

```csharp
await Parallel.ForEachAsync(urls,
    new ParallelOptions { MaxDegreeOfParallelism = 10, CancellationToken = ct },
    async (url, token) => {
        var data = await httpClient.GetStringAsync(url, token);
        await ProcessAsync(data, token);
    });
```

`Parallel.For` / `ForEach` are SYNC — they call the body synchronously. For ASYNC work (HTTP, I/O), the equivalent is `Parallel.ForEachAsync` — added in .NET 6.

This is **the** modern API for "do these N async operations with controlled concurrency." Cleaner than rolling your own SemaphoreSlim + WhenAll.

```csharp
// Old pattern
using var sem = new SemaphoreSlim(10);
var tasks = urls.Select(async url => {
    await sem.WaitAsync();
    try { await DownloadAsync(url); }
    finally { sem.Release(); }
});
await Task.WhenAll(tasks);

// Modern equivalent
await Parallel.ForEachAsync(urls,
    new ParallelOptions { MaxDegreeOfParallelism = 10 },
    async (url, ct) => await DownloadAsync(url, ct));
```

---

## When to use which

| Workload | Use |
|---|---|
| CPU-bound, index-based | `Parallel.For` |
| CPU-bound, collection-based | `Parallel.ForEach` |
| Async I/O with controlled concurrency | `Parallel.ForEachAsync` |
| Functional pipeline (filter, project, aggregate) | PLINQ |
| Long-running CPU work as single task | `Task.Run` |
| Producer-consumer | `Channel<T>` |

---

## PLINQ — Parallel LINQ

```csharp
var result = items.AsParallel()
    .Where(x => Heavy(x))
    .Select(x => Transform(x))
    .Sum();
```

`AsParallel()` switches the rest of the LINQ chain to run in parallel. The runtime partitions the source, runs the operators on multiple threads, recombines results.

### When PLINQ wins

- The work per item is **expensive** (else partition overhead dominates).
- The source is large.
- The operations are **stateless** (no side effects).

### When it loses

- Items are cheap to process — overhead beats parallelism.
- The collection is small.
- The aggregation is order-dependent (PLINQ might shuffle order).

For "small loop with cheap body," regular LINQ wins. For "10K items, each heavy computation," PLINQ wins.

### Options

```csharp
items.AsParallel()
    .WithDegreeOfParallelism(4)
    .WithCancellation(ct)
    .WithExecutionMode(ParallelExecutionMode.ForceParallelism)
    .Where(x => Heavy(x))
    .ToList();
```

`ForceParallelism` overrides the heuristic that might decide a workload isn't worth parallelizing.

---

## Order preservation

PLINQ may shuffle results unless you ask for order:

```csharp
items.AsParallel().Select(...).ToList();   // unordered
items.AsParallel().AsOrdered().Select(...).ToList();   // ordered (slight overhead)
```

For `Parallel.For`, the iterations happen in **any order** but each produces a known index. For most patterns, order doesn't matter (e.g., aggregating sums) — leave it unordered for speed.

---

## Cancellation in Parallel

```csharp
using var cts = new CancellationTokenSource();
try {
    Parallel.For(0, 1_000_000, new ParallelOptions { CancellationToken = cts.Token }, i => {
        cts.Token.ThrowIfCancellationRequested();   // optional inner check
        Compute(i);
    });
} catch (OperationCanceledException) {
    // canceled
}
```

When the token cancels, `Parallel.For` waits for in-progress iterations to finish, then throws OperationCanceledException.

For early exit, throw `OperationCanceledException` from the body — or use `loopState.Break() / Stop()`:

```csharp
Parallel.For(0, 1_000_000, (i, loopState) => {
    if (Found(i)) {
        result = i;
        loopState.Stop();   // signal early termination
    }
});
```

`Break` lets in-progress lower-index iterations finish; `Stop` is aggressive.

---

## Exceptions in Parallel

Parallel loops aggregate exceptions:

```csharp
try {
    Parallel.For(0, 100, i => {
        if (i == 50) throw new InvalidOperationException("fail");
        if (i == 75) throw new ArgumentException("other fail");
    });
} catch (AggregateException ex) {
    foreach (var inner in ex.InnerExceptions) {
        Console.WriteLine(inner.Message);
    }
}
```

If any iteration throws, the loop completes the rest and bundles errors into an AggregateException.

For `Parallel.ForEachAsync` since .NET 6, exceptions are still aggregated but presented through `await` (first one wins; rest available via the Task's Exception property).

---

## CPU vs I/O — DON'T mix Parallel.For with async

```csharp
Parallel.For(0, urls.Length, async i => {     // ⚠ — async lambda with sync Parallel.For
    var data = await httpClient.GetStringAsync(urls[i]);
    // ...
});
```

`Parallel.For` doesn't `await` the lambda's task. It just calls the lambda, sees a Task return, and considers the iteration "done" instantly — even though the work is still happening.

For async work, use `Parallel.ForEachAsync` (.NET 6+) instead.

---

## Long-running tasks vs Parallel

```csharp
Task.Factory.StartNew(LongJob, TaskCreationOptions.LongRunning);
```

`LongRunning` tells the scheduler "this task will run for a long time — don't use a thread pool thread, create a dedicated one." Useful for game loops, monitor threads.

For shorter, embarrassingly-parallel work, prefer `Task.Run` or `Parallel.For` — they use the thread pool efficiently.

---

## TPL Dataflow

`System.Threading.Tasks.Dataflow` is a separate library for **complex pipelines**:

```csharp
var transform = new TransformBlock<int, string>(async i => (await DownloadAsync(i)).Length);
var action = new ActionBlock<string>(s => Process(s));
transform.LinkTo(action, new DataflowLinkOptions { PropagateCompletion = true });

// Send work
foreach (var i in items) await transform.SendAsync(i);
transform.Complete();
await action.Completion;
```

Blocks: TransformBlock, ActionBlock, BroadcastBlock, BufferBlock, BatchBlock, JoinBlock, etc. Each block has its own concurrency settings.

Used for high-throughput data pipelines. More complex than Channels but more featureful for branching/merging dataflow.

For most code, Channels are simpler and sufficient.

---

## Internals — how Parallel.For partitions

The TPL uses a **work-stealing partitioner** by default:

1. Splits the index range into chunks (initially equal-sized per core).
2. Each thread pool worker gets a chunk.
3. When a worker finishes its chunk, it **steals** from others' remaining chunks.
4. Self-balancing: fast threads pick up slack from slow ones.

For uneven workloads (some indices are expensive, others cheap), work-stealing keeps all cores busy.

You can customize the partitioner:
```csharp
Parallel.For(Partitioner.Create(0, n, chunkSize: 100), range => {
    for (int i = range.Item1; i < range.Item2; i++) Compute(i);
});
```

Useful for hot loops where you want to control granularity.

### Why thread-local state matters

Without thread-local aggregation:
```csharp
long sum = 0;
Parallel.For(0, n, i => Interlocked.Add(ref sum, i));  // ⚠ contention on every iteration
```

Every iteration hits the same atomic — false sharing, cache-line bouncing, throughput collapse.

With thread-local:
```csharp
long sum = 0;
Parallel.For(0, n,
    () => 0L,
    (i, _, local) => local + i,
    final => Interlocked.Add(ref sum, final));
// One atomic per thread instead of per iteration
```

For tight loops, this is the difference between 10× faster and slower than sequential.

---

## Common patterns

### Map-Reduce

```csharp
// Map: parallel transform
// Reduce: aggregate

var transformed = items.AsParallel().Select(Transform).ToList();
var total = transformed.Sum();

// Or in one pass with thread-local state
long total = 0;
Parallel.For(0, items.Count,
    () => 0L,
    (i, _, local) => local + Transform(items[i]),
    final => Interlocked.Add(ref total, final));
```

### Parallel matrix multiply

```csharp
Parallel.For(0, rows, i => {
    for (int j = 0; j < cols; j++) {
        double sum = 0;
        for (int k = 0; k < n; k++) sum += a[i, k] * b[k, j];
        c[i, j] = sum;
    }
});
```

Parallelize the outer loop. Inner loops stay sequential for cache efficiency.

### Concurrent HTTP with throttling

```csharp
await Parallel.ForEachAsync(urls,
    new ParallelOptions { MaxDegreeOfParallelism = 20, CancellationToken = ct },
    async (url, ct) => {
        var data = await httpClient.GetStringAsync(url, ct);
        results[url] = Parse(data);
    });
```

Modern replacement for SemaphoreSlim + WhenAll.

---

## When TPL is overkill

- For 100 items with cheap body — sequential is faster (overhead beats parallel).
- For async I/O with no CPU — Task.WhenAll alone is enough, or Parallel.ForEachAsync.
- For one-off work — Task.Run is simpler.

Profile to know if parallelism actually helps. For trivially-parallel CPU work, gain ≈ core count. For barely-CPU-bound or I/O-bound, gain is much less.

---

## Common bugs

### Mutating shared state without sync

```csharp
List<int> results = new();
Parallel.For(0, n, i => results.Add(Compute(i)));   // ⚠ List<T> isn't thread-safe
```

Use `ConcurrentBag<T>` or `ConcurrentQueue<T>`. Or accumulate per-thread and merge.

### Capturing the loop variable

```csharp
var tasks = new List<Task>();
for (int i = 0; i < 10; i++) {
    tasks.Add(Task.Run(() => Process(i)));   // ⚠ captures i — all see final value
}
```

Same as the classic for-loop closure trap. Fix:
```csharp
for (int i = 0; i < 10; i++) {
    int copy = i;
    tasks.Add(Task.Run(() => Process(copy)));
}
```

Or `Parallel.For(0, 10, i => Process(i));` — each iteration has its own i.

### Exception swallowed in Task.Run

```csharp
Task.Run(() => DoSomething());   // not awaited
// If DoSomething throws, the exception is lost (or surfaced via UnobservedTaskException eventually)
```

Always await or `_ = Task.Run(...)` with internal try/catch + logging.

### Async inside Parallel.For

```csharp
Parallel.For(0, 100, async i => { await DownloadAsync(i); });   // ⚠ doesn't await
```

Use `Parallel.ForEachAsync`.

### Wrong default MaxDegreeOfParallelism

Default is **unlimited** (well, ProcessorCount-ish). For I/O-bound work, that might mean hundreds of concurrent operations slamming the network or DB. Set MaxDegreeOfParallelism explicitly.

---

## Performance summary

- Parallel.For overhead per iteration: ~microsecond for the dispatch.
- Speedup: up to N× for N cores on embarrassingly parallel work.
- For trivial loops (<1 ms total), parallel can be slower than sequential.
- PLINQ: similar, with additional partitioning overhead.

Profile. Measure. Don't assume parallel = fast.

---

## When to use TPL vs alternatives

| Need | Use |
|---|---|
| CPU-bound loop (sync) | `Parallel.For` / `Parallel.ForEach` |
| Async loop with concurrency limit | `Parallel.ForEachAsync` |
| Stream operations on collections | PLINQ |
| Producer-consumer | `Channel<T>` |
| Complex dataflow graph | TPL Dataflow |
| Single CPU task | `Task.Run` |
| Many short async I/O | `Task.WhenAll` |

---

## Summary

- TPL is for CPU-bound parallelism.
- `Parallel.For` / `ForEach` for sync data parallelism.
- `Parallel.ForEachAsync` (.NET 6+) for async work with throttling.
- PLINQ for functional-pipeline-style parallelism.
- Use thread-local state for aggregation — avoid sync on every iteration.
- For I/O, prefer async/await + Task.WhenAll over Parallel.

→ Next: [15-TaskWhenAllWhenAny.md](15-TaskWhenAllWhenAny.md)
