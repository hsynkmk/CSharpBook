# Chapter 08 — Coding Problems

> 20 hands-on concurrency problems. The most important chapter's coding section — every senior C# interview tests these.

---

## Problem 1 — Thread-safe counter

Implement a counter that supports `Increment()` and `Read()` safely across threads. Three ways: lock, Interlocked, ConcurrentDictionary-style.

<details><summary>Solution</summary>

```csharp
// 1. lock
public class Counter1 {
    private readonly object _lock = new();
    private int _value;
    public void Increment() { lock (_lock) _value++; }
    public int Read() { lock (_lock) return _value; }
}

// 2. Interlocked — fastest
public class Counter2 {
    private int _value;
    public void Increment() => Interlocked.Increment(ref _value);
    public int Read() => Volatile.Read(ref _value);
}

// 3. ConcurrentDictionary-style (silly here, but illustrative)
public class Counter3 {
    private readonly ConcurrentDictionary<int, int> _dict = new() { [0] = 0 };
    public void Increment() => _dict.AddOrUpdate(0, 1, (_, v) => v + 1);
    public int Read() => _dict[0];
}
```

For a simple counter, Interlocked wins on speed. The lock version is clearer but slower under contention.

</details>

---

## Problem 2 — Predict the deadlock

```csharp
public static int Main() {
    return GetAsync().Result;
}
public static async Task<int> GetAsync() {
    await Task.Delay(100);
    return 42;
}
```

In a console app — works? Deadlock? Why?

<details><summary>Answer</summary>

**Works.** Console apps have NO SynchronizationContext. After `Task.Delay`, the continuation runs on any free thread pool thread. The main thread blocks on `.Result`; eventually the task completes and main resumes.

If this were a WPF app with the same code in a button handler, **deadlock** — the captured UI SynchronizationContext is needed for the continuation, but the UI thread is blocked.

Modern .NET console apps: use `static async Task<int> Main()` instead.

</details>

---

## Problem 3 — Build a throttled downloader

Download N URLs concurrently, max 5 at a time, with 10-second per-request timeout.

<details><summary>Solution</summary>

```csharp
public async Task<string[]> DownloadAllAsync(IList<string> urls, CancellationToken outerCt = default) {
    using var client = new HttpClient();
    var results = new string[urls.Count];

    await Parallel.ForEachAsync(
        urls.Select((url, idx) => (url, idx)),
        new ParallelOptions { MaxDegreeOfParallelism = 5, CancellationToken = outerCt },
        async (item, token) => {
            using var timeout = CancellationTokenSource.CreateLinkedTokenSource(token);
            timeout.CancelAfter(TimeSpan.FromSeconds(10));
            try {
                results[item.idx] = await client.GetStringAsync(item.url, timeout.Token);
            } catch (OperationCanceledException) when (!token.IsCancellationRequested) {
                results[item.idx] = "<timeout>";
            }
        });

    return results;
}
```

`Parallel.ForEachAsync` handles throttling. Linked CTS combines outer cancellation with per-request timeout. The `when` filter distinguishes timeout from outer cancellation.

</details>

---

## Problem 4 — Producer-consumer with Channel

Implement a background worker queue: enqueue jobs; one consumer task processes them sequentially.

<details><summary>Solution</summary>

```csharp
public class JobQueue {
    private readonly Channel<Func<CancellationToken, Task>> _channel =
        Channel.CreateBounded<Func<CancellationToken, Task>>(new BoundedChannelOptions(1000) {
            SingleReader = true,
            SingleWriter = false,
        });

    public ValueTask EnqueueAsync(Func<CancellationToken, Task> job, CancellationToken ct = default) =>
        _channel.Writer.WriteAsync(job, ct);

    public async Task RunAsync(CancellationToken ct) {
        try {
            await foreach (var job in _channel.Reader.ReadAllAsync(ct)) {
                try { await job(ct); }
                catch (Exception ex) { /* log */ }
            }
        } finally {
            _channel.Writer.TryComplete();
        }
    }
}

// Usage
var queue = new JobQueue();
_ = queue.RunAsync(appShutdownToken);
await queue.EnqueueAsync(async ct => await SendEmailAsync(ct));
```

`SingleReader = true` enables internal optimization. Per-job exception isolation. Bounded channel provides backpressure.

</details>

---

## Problem 5 — Atomic snapshot + update with immutable data

Implement a thread-safe key-value store using `ImmutableDictionary` + Interlocked.

<details><summary>Solution</summary>

```csharp
public class AtomicStore<TKey, TValue> where TKey : notnull {
    private ImmutableDictionary<TKey, TValue> _state = ImmutableDictionary<TKey, TValue>.Empty;

    public TValue? Get(TKey key) {
        var snapshot = _state;   // single atomic read
        return snapshot.TryGetValue(key, out var v) ? v : default;
    }

    public void Set(TKey key, TValue value) {
        ImmutableDictionary<TKey, TValue> current, updated;
        do {
            current = _state;
            updated = current.SetItem(key, value);
        } while (Interlocked.CompareExchange(ref _state, updated, current) != current);
    }

    // Or with ImmutableInterlocked
    public void Set2(TKey key, TValue value) =>
        ImmutableInterlocked.Update(ref _state, c => c.SetItem(key, value));
}
```

Readers never lock. Writers do a CAS loop with the updated copy. For high-contention writes, retries can pile up — but no deadlocks, ever.

</details>

---

## Problem 6 — Find the async bug

```csharp
public class Service {
    private readonly List<int> _results = new();

    public async Task ProcessAsync(IEnumerable<int> items) {
        await Parallel.ForEachAsync(items, async (item, ct) => {
            var r = await ComputeAsync(item, ct);
            _results.Add(r);   // ⚠
        });
    }
}
```

<details><summary>Answer</summary>

`List<T>` isn't thread-safe. Concurrent Adds can corrupt the internal array, lose items, or throw.

Fix:
```csharp
private readonly ConcurrentBag<int> _results = new();   // thread-safe
```

Or:
```csharp
var bag = new ConcurrentBag<int>();
await Parallel.ForEachAsync(items, async (item, ct) => {
    var r = await ComputeAsync(item, ct);
    bag.Add(r);
});
_results = bag.ToList();
```

Or use `Parallel.ForEachAsync` with state aggregation, then merge once at the end.

</details>

---

## Problem 7 — Implement a SemaphoreSlim from scratch

Build `MySemaphore` with `WaitAsync` and `Release`, using `TaskCompletionSource` for waiters.

<details><summary>Solution</summary>

```csharp
public class MySemaphore {
    private int _currentCount;
    private readonly int _maxCount;
    private readonly Queue<TaskCompletionSource<bool>> _waiters = new();
    private readonly object _lock = new();

    public MySemaphore(int initialCount, int maxCount) {
        _currentCount = initialCount;
        _maxCount = maxCount;
    }

    public Task WaitAsync(CancellationToken ct = default) {
        lock (_lock) {
            if (_currentCount > 0) {
                _currentCount--;
                return Task.CompletedTask;
            }
            var tcs = new TaskCompletionSource<bool>(TaskCreationOptions.RunContinuationsAsynchronously);
            _waiters.Enqueue(tcs);
            if (ct.CanBeCanceled) {
                ct.Register(() => tcs.TrySetCanceled(ct));
            }
            return tcs.Task;
        }
    }

    public void Release() {
        lock (_lock) {
            if (_waiters.Count > 0) {
                while (_waiters.Count > 0) {
                    var w = _waiters.Dequeue();
                    if (w.TrySetResult(true)) return;
                    // else canceled; try next
                }
            }
            if (_currentCount >= _maxCount) throw new SemaphoreFullException();
            _currentCount++;
        }
    }
}
```

Real SemaphoreSlim has more optimizations (lock-free fast paths, etc.) but the structure is the same. The `RunContinuationsAsynchronously` flag avoids running continuations on the releasing thread.

</details>

---

## Problem 8 — Cancellable long-running computation

A method does CPU-heavy work in a loop. Add cancellation support without too much overhead.

<details><summary>Solution</summary>

```csharp
public int Compute(int[] data, CancellationToken ct = default) {
    int sum = 0;
    for (int i = 0; i < data.Length; i++) {
        if ((i & 0xFFFF) == 0)   // check every 65536 iterations
            ct.ThrowIfCancellationRequested();
        sum += Heavy(data[i]);
    }
    return sum;
}
```

Checking every iteration adds branch overhead; checking every 65536 is essentially free per iteration but still responsive enough (~1ms-per-check for typical work).

For very-fast loops, check less frequently. For slow inner work, check every iteration is fine.

</details>

---

## Problem 9 — LRU cache (thread-safe)

Implement a thread-safe LRU cache. Hint: ConcurrentDictionary + LinkedList won't quite work; consider using a lock.

<details><summary>Solution</summary>

```csharp
public class ThreadSafeLruCache<TKey, TValue> where TKey : notnull {
    private readonly int _capacity;
    private readonly object _lock = new();
    private readonly Dictionary<TKey, LinkedListNode<(TKey K, TValue V)>> _map = new();
    private readonly LinkedList<(TKey K, TValue V)> _order = new();

    public ThreadSafeLruCache(int capacity) { _capacity = capacity; }

    public bool TryGet(TKey key, out TValue value) {
        lock (_lock) {
            if (_map.TryGetValue(key, out var node)) {
                _order.Remove(node);
                _order.AddFirst(node);
                value = node.Value.V;
                return true;
            }
            value = default!;
            return false;
        }
    }

    public void Set(TKey key, TValue value) {
        lock (_lock) {
            if (_map.TryGetValue(key, out var existing)) {
                _order.Remove(existing);
                existing.Value = (key, value);
                _order.AddFirst(existing);
                return;
            }
            if (_map.Count == _capacity) {
                var oldest = _order.Last!;
                _order.RemoveLast();
                _map.Remove(oldest.Value.K);
            }
            var node = _order.AddFirst((key, value));
            _map[key] = node;
        }
    }
}
```

The lock is global; for very-high contention, consider sharding (multiple sub-caches by hash). But this works correctly.

`ConcurrentDictionary` alone won't do — LRU eviction needs ordering info that requires lock-step access between the dict and the order list.

</details>

---

## Problem 10 — Spot the deadlock

```csharp
public class Bank {
    private readonly Dictionary<int, Account> _accounts = new();
    public void Transfer(int from, int to, decimal amount) {
        lock (_accounts[from]) {
            lock (_accounts[to]) {
                _accounts[from].Balance -= amount;
                _accounts[to].Balance += amount;
            }
        }
    }
}
```

What's the deadlock? Fix it.

<details><summary>Answer</summary>

Thread 1: `Transfer(A, B, ...)` — locks A, waits for B.
Thread 2: `Transfer(B, A, ...)` — locks B, waits for A.
**Deadlock.**

**Fix**: always acquire locks in a canonical order (e.g., lower ID first):

```csharp
public void Transfer(int from, int to, decimal amount) {
    var (firstId, secondId) = from < to ? (from, to) : (to, from);
    lock (_accounts[firstId]) {
        lock (_accounts[secondId]) {
            _accounts[from].Balance -= amount;
            _accounts[to].Balance += amount;
        }
    }
}
```

Now all transfers acquire locks in ID order. No two threads can hold locks in opposite directions.

Also: lock on dedicated objects, not the Account itself. And consider Interlocked or atomic decimal libraries for the actual update.

</details>

---

## Problem 11 — Cancel-on-first-failure

Run N async operations; if any throws, cancel the rest and rethrow.

<details><summary>Solution</summary>

```csharp
public async Task RunAllOrFailFastAsync(
    IEnumerable<Func<CancellationToken, Task>> ops,
    CancellationToken outerCt = default)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(outerCt);
    var token = cts.Token;

    var tasks = ops.Select(async op => {
        try {
            await op(token);
        } catch (Exception) when (!token.IsCancellationRequested) {
            cts.Cancel();
            throw;
        }
    });

    await Task.WhenAll(tasks);
}
```

When any operation throws (and it wasn't already canceled), we trigger cancellation. Other tasks observe the token, throw their own OCEs (or finish cleanly). WhenAll waits for everyone, then surfaces the original exception.

</details>

---

## Problem 12 — Implement Channel<T> from scratch (toy version)

A minimal Channel: WriteAsync, ReadAsync, Complete. Don't worry about boundedness or performance.

<details><summary>Solution</summary>

```csharp
public class ToyChannel<T> {
    private readonly Queue<T> _buffer = new();
    private readonly Queue<TaskCompletionSource<T>> _waiters = new();
    private bool _complete;
    private readonly object _lock = new();

    public Task WriteAsync(T item) {
        lock (_lock) {
            if (_complete) throw new InvalidOperationException("complete");
            if (_waiters.Count > 0) {
                while (_waiters.Count > 0) {
                    var w = _waiters.Dequeue();
                    if (w.TrySetResult(item)) return Task.CompletedTask;
                }
            }
            _buffer.Enqueue(item);
            return Task.CompletedTask;
        }
    }

    public Task<T> ReadAsync(CancellationToken ct = default) {
        lock (_lock) {
            if (_buffer.Count > 0) return Task.FromResult(_buffer.Dequeue());
            if (_complete) throw new InvalidOperationException("complete");
            var tcs = new TaskCompletionSource<T>(TaskCreationOptions.RunContinuationsAsynchronously);
            _waiters.Enqueue(tcs);
            if (ct.CanBeCanceled) ct.Register(() => tcs.TrySetCanceled(ct));
            return tcs.Task;
        }
    }

    public void Complete() {
        lock (_lock) {
            _complete = true;
            while (_waiters.Count > 0) _waiters.Dequeue().TrySetException(new InvalidOperationException("closed"));
        }
    }
}
```

Real Channels are far more sophisticated (bounded variants with backpressure, lock-free fast paths, ValueTask, etc.). But this captures the core idea.

</details>

---

## Problem 13 — Async retry with exponential backoff

Build a `RetryAsync<T>` that retries up to N times with exponential backoff.

<details><summary>Solution</summary>

```csharp
public async Task<T> RetryAsync<T>(
    Func<CancellationToken, Task<T>> op,
    int maxAttempts = 3,
    TimeSpan initialDelay = default,
    CancellationToken ct = default)
{
    if (initialDelay == default) initialDelay = TimeSpan.FromMilliseconds(100);
    Exception? lastEx = null;

    for (int attempt = 0; attempt < maxAttempts; attempt++) {
        try {
            return await op(ct);
        } catch (OperationCanceledException) when (ct.IsCancellationRequested) {
            throw;   // outer cancellation — propagate immediately
        } catch (Exception ex) {
            lastEx = ex;
            if (attempt == maxAttempts - 1) break;
            var delay = TimeSpan.FromTicks(initialDelay.Ticks * (1L << attempt));   // 100ms, 200ms, 400ms, 800ms...
            await Task.Delay(delay, ct);
        }
    }

    throw new AggregateException("All retries failed", lastEx!);
}

// Usage
var result = await RetryAsync(ct => httpClient.GetStringAsync(url, ct), maxAttempts: 3);
```

Real-world: use `Microsoft.Extensions.Resilience` or Polly with jitter, circuit breakers, etc.

</details>

---

## Problem 14 — Find the bug

```csharp
public class Cache {
    private readonly ConcurrentDictionary<string, User> _cache = new();
    public User GetOrLoad(string key) {
        if (!_cache.ContainsKey(key)) {
            var user = LoadFromDb(key);
            _cache[key] = user;
        }
        return _cache[key];
    }
}
```

<details><summary>Answer</summary>

Two issues:

1. **Race**: between `ContainsKey` and `_cache[key] = user`, another thread might have loaded the same key. Two `LoadFromDb` calls for the same key — wasteful (or buggy if DB call has side effects).

2. **`_cache[key]` after Set could throw** if the dict somehow has the key removed between Set and Get (rare but possible).

Fix:
```csharp
public User GetOrLoad(string key) =>
    _cache.GetOrAdd(key, k => LoadFromDb(k));
```

But beware: GetOrAdd's factory may run multiple times under contention. To guarantee one-time load, use Lazy:

```csharp
private readonly ConcurrentDictionary<string, Lazy<User>> _cache = new();
public User GetOrLoad(string key) {
    var lazy = _cache.GetOrAdd(key, k => new Lazy<User>(() => LoadFromDb(k)));
    return lazy.Value;
}
```

Lazy guarantees the factory runs exactly once.

</details>

---

## Problem 15 — Hedge two requests

Make two HTTP requests; return whichever finishes first.

<details><summary>Solution</summary>

```csharp
public async Task<string> HedgeAsync(string primary, string backup, CancellationToken ct = default) {
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    var primaryTask = _http.GetStringAsync(primary, cts.Token);
    var backupTask = _http.GetStringAsync(backup, cts.Token);

    var winner = await Task.WhenAny(primaryTask, backupTask);
    cts.Cancel();   // cancel the loser

    try { return await winner; }
    catch when (winner == backupTask && !primaryTask.IsFaulted) {
        return await primaryTask;   // fall back if backup failed but primary still working — unlikely
    }
}
```

`WhenAny` waits for either; we cancel the loser. The exception handling tries to fall back if the winner unexpectedly threw.

For real hedging, use `Microsoft.Extensions.Resilience.AddHedgingHandler` — handles all the edge cases.

</details>

---

## Problem 16 — Async stream + cancellation

Stream lines from a URL, with consumer-side cancellation.

<details><summary>Solution</summary>

```csharp
public async IAsyncEnumerable<string> ReadLinesAsync(
    string url,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    using var client = new HttpClient();
    using var resp = await client.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, ct);
    using var stream = await resp.Content.ReadAsStreamAsync(ct);
    using var reader = new StreamReader(stream);

    string? line;
    while ((line = await reader.ReadLineAsync(ct)) is not null) {
        yield return line;
    }
}

// Consumer
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
await foreach (var line in ReadLinesAsync(url).WithCancellation(cts.Token)) {
    Console.WriteLine(line);
}
```

`[EnumeratorCancellation]` wires `WithCancellation` to the iterator's token. The iterator forwards the token to every inner async call. Cancellation propagates cleanly.

</details>

---

## Problem 17 — Producer-consumer with fan-out

One producer reads from a source. Multiple consumers process in parallel. Use Channels.

<details><summary>Solution</summary>

```csharp
public async Task ProcessAsync(IEnumerable<Item> source, int workerCount, CancellationToken ct) {
    var channel = Channel.CreateBounded<Item>(new BoundedChannelOptions(100) {
        SingleWriter = true,
        SingleReader = false,
    });

    var producer = Task.Run(async () => {
        try {
            foreach (var item in source) {
                await channel.Writer.WriteAsync(item, ct);
            }
        } finally {
            channel.Writer.Complete();
        }
    }, ct);

    var consumers = Enumerable.Range(0, workerCount).Select(_ => Task.Run(async () => {
        await foreach (var item in channel.Reader.ReadAllAsync(ct)) {
            await ProcessItemAsync(item, ct);
        }
    }, ct));

    await Task.WhenAll(consumers.Append(producer));
}
```

`Complete()` in `finally` ensures consumers exit even on exception. `Task.WhenAll` waits for everyone.

</details>

---

## Problem 18 — Lock vs Interlocked benchmark

For a hot counter incremented from 4 threads, predict relative performance:
- a) `lock (gate) _counter++`
- b) `Interlocked.Increment(ref _counter)`
- c) Per-thread counters merged at the end

<details><summary>Answer</summary>

Rough benchmark (4 threads, 10M increments each):

- a) lock: ~5-10 seconds (heavy contention on the lock).
- b) Interlocked: ~1-2 seconds (atomic, but cache-line bouncing).
- c) Per-thread + merge: ~0.2 seconds (no inter-thread coordination during the hot loop).

For raw counters, per-thread aggregation always wins under contention. Pattern:

```csharp
long[] perThread = new long[Environment.ProcessorCount];
Parallel.For(0, n,
    () => 0L,
    (i, _, local) => local + 1,
    final => Interlocked.Add(ref perThread[Thread.CurrentThread.ManagedThreadId % perThread.Length], final));
long total = perThread.Sum();
```

Or use `Parallel.For` with thread-local state directly.

</details>

---

## Problem 19 — Worker service with graceful shutdown

A background service that pulls from a queue and processes items. Shuts down cleanly when canceled.

<details><summary>Solution</summary>

```csharp
public class Worker {
    private readonly Channel<Job> _channel = Channel.CreateBounded<Job>(100);
    private Task? _runner;

    public void Start(CancellationToken stopToken) {
        _runner = Task.Run(async () => {
            await foreach (var job in _channel.Reader.ReadAllAsync(stopToken)) {
                try { await job.RunAsync(stopToken); }
                catch (Exception ex) { _log.LogError(ex, "job failed"); }
            }
        }, stopToken);
    }

    public ValueTask EnqueueAsync(Job job, CancellationToken ct = default) =>
        _channel.Writer.WriteAsync(job, ct);

    public async Task StopAsync() {
        _channel.Writer.TryComplete();   // signal no more
        if (_runner is not null) await _runner;
    }
}
```

`stopToken` lets the worker exit its loop early. `StopAsync` finishes pending items first (because we don't cancel; we just complete the writer).

In ASP.NET Core, use `BackgroundService` — handles startup/shutdown integration with the host.

</details>

---

## Problem 20 — Stress test: write and read concurrently

Test that a `ConcurrentDictionary<int, string>` behaves correctly under concurrent reads/writes from 10 threads.

<details><summary>Solution sketch</summary>

```csharp
var dict = new ConcurrentDictionary<int, string>();
var rng = new Random();

var writers = Enumerable.Range(0, 5).Select(t => Task.Run(() => {
    for (int i = 0; i < 100_000; i++) {
        int key = rng.Next(100);
        dict[key] = $"v{key}-{i}";
    }
}));

var readers = Enumerable.Range(0, 5).Select(t => Task.Run(() => {
    for (int i = 0; i < 100_000; i++) {
        int key = rng.Next(100);
        dict.TryGetValue(key, out _);
    }
}));

await Task.WhenAll(writers.Concat(readers));
Console.WriteLine($"final count: {dict.Count}");
```

ConcurrentDictionary handles this fine. With `Dictionary<int, string>` + no lock, you'd see corruption (random crashes, infinite loops in lookup).

For property-based testing, libraries like FsCheck can generate concurrent test scenarios systematically.

</details>

---

## Summary

You've now drilled:
- Threads vs Tasks.
- async/await mental model.
- Cancellation propagation.
- Locking primitives (lock, SemaphoreSlim, Interlocked).
- Concurrent collections.
- Channels for producer-consumer.
- TPL for CPU-bound parallelism.
- WhenAll / WhenAny / WhenEach.
- Common async bugs and how to avoid them.

These are the patterns of production concurrent code. Internalize them.

→ [Chapter 09 — Memory & Performance](../09-MemoryPerformance/README.md)
