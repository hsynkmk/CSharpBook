# Semaphores — SemaphoreSlim

## What it is

A **semaphore** is a counter-based concurrency primitive. It has N "permits"; threads acquire one before entering the critical section and release one when done. When all permits are taken, further acquires block (or await) until someone releases.

```csharp
private readonly SemaphoreSlim _sem = new(initialCount: 3, maxCount: 3);

public async Task DownloadAsync(string url) {
    await _sem.WaitAsync();
    try {
        await _http.GetAsync(url);   // at most 3 concurrent downloads
    } finally {
        _sem.Release();
    }
}
```

`SemaphoreSlim` is the **async-friendly** semaphore — its `WaitAsync` doesn't block a thread. Critical for async code.

`Semaphore` (without "Slim") is the older, kernel-level variant — slower, blocks threads, supports cross-process. Use `SemaphoreSlim` unless you need cross-process.

---

## Why it exists

Two big use cases:

### 1. Throttling concurrent operations

"At most N requests to the database at once." "At most 10 concurrent HTTP calls." "Limit to 4 parallel file uploads."

A `lock` would allow only 1. A semaphore with N permits allows N.

### 2. Async mutual exclusion

`lock` can't span `await`. `SemaphoreSlim(1, 1)` (one permit) is the **async-safe mutex**:

```csharp
private readonly SemaphoreSlim _gate = new(1, 1);

public async Task M() {
    await _gate.WaitAsync();
    try {
        await DoWorkAsync();   // safe — await is fine inside semaphore
    } finally {
        _gate.Release();
    }
}
```

This is the modern "I need a mutex but my code is async" pattern. Universal.

---

## Basics

### Construction

```csharp
new SemaphoreSlim(initialCount);
new SemaphoreSlim(initialCount, maxCount);
```

- `initialCount` — initial permits available.
- `maxCount` — upper bound on permits. Optional; defaults to int.MaxValue.

For a binary semaphore (mutex): `new SemaphoreSlim(1, 1)`.
For throttling: `new SemaphoreSlim(N, N)`.

### Acquiring

```csharp
await _sem.WaitAsync();                                  // async — doesn't block thread
await _sem.WaitAsync(cancellationToken);                  // cancellable
await _sem.WaitAsync(TimeSpan.FromSeconds(5));             // timeout — returns bool
bool got = await _sem.WaitAsync(timeout, cancellationToken);

_sem.Wait();                                              // sync — blocks thread (rare)
```

Always pair with `try/finally`:

```csharp
await _sem.WaitAsync(ct);
try {
    // critical section
} finally {
    _sem.Release();
}
```

### Releasing

```csharp
_sem.Release();           // release one permit
_sem.Release(5);          // release 5 permits at once (rare)
```

**Critical**: every `Wait` must be matched by a `Release` (or the semaphore's permit count slowly drains). Use `try/finally` to guarantee.

### Current count

```csharp
int available = _sem.CurrentCount;   // approximation — value can change immediately
```

Snapshot. Rarely used directly; race-prone if you decide based on it.

---

## Throttling — the canonical pattern

```csharp
public class DownloadService {
    private readonly SemaphoreSlim _throttle = new(5);   // max 5 concurrent

    public async Task<string> DownloadAsync(string url, CancellationToken ct = default) {
        await _throttle.WaitAsync(ct);
        try {
            return await _http.GetStringAsync(url, ct);
        } finally {
            _throttle.Release();
        }
    }
}

// 100 concurrent calls — only 5 active at any time
var tasks = urls.Select(u => svc.DownloadAsync(u));
var results = await Task.WhenAll(tasks);
```

100 tasks start; 5 grab permits; the other 95 await. As each completes, the next gets a permit. Same fan-out shape, controlled concurrency.

Modern alternative: `Parallel.ForEachAsync` with `MaxDegreeOfParallelism = 5` (see [§14 TPL](14-TPL.md)). Both work; the semaphore form is more flexible for nested code.

---

## Async mutex pattern

```csharp
public class Cache<T> {
    private readonly SemaphoreSlim _lock = new(1, 1);
    private T? _value;
    private bool _loaded;

    public async Task<T> GetAsync(Func<Task<T>> loader, CancellationToken ct = default) {
        if (_loaded) return _value!;
        await _lock.WaitAsync(ct);
        try {
            if (_loaded) return _value!;   // double-check
            _value = await loader();
            _loaded = true;
            return _value;
        } finally {
            _lock.Release();
        }
    }
}
```

Async double-checked locking. One concurrent loader; subsequent callers wait, then see the cached value.

Note: `Lazy<Task<T>>` is often a cleaner solution for this specific case (lazy async init).

---

## `WaitAsync` cancellation

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
try {
    await _sem.WaitAsync(cts.Token);
} catch (OperationCanceledException) {
    Console.WriteLine("Couldn't get a permit in 5s");
}
```

If the token cancels before a permit is available, throws `OperationCanceledException`. The semaphore is NOT acquired — no need to release.

`WaitAsync(TimeSpan)` returns `bool`:

```csharp
if (await _sem.WaitAsync(TimeSpan.FromSeconds(5))) {
    try { /* ... */ }
    finally { _sem.Release(); }
} else {
    Console.WriteLine("timeout");
}
```

---

## Sync vs async API

```csharp
_sem.Wait();           // sync — blocks the thread
await _sem.WaitAsync(); // async — releases the thread to the pool
```

In async code, **always use `WaitAsync`**. `Wait` blocks the thread, defeating async's purpose.

In purely sync code, `Wait` is fine. But if any caller of yours might be async, expose `WaitAsync`.

---

## Common bugs

### Forgetting Release on exception

```csharp
await _sem.WaitAsync();
DoStuff();   // throws
_sem.Release();   // ⚠ never reached
```

Use try/finally. Always.

```csharp
await _sem.WaitAsync();
try { DoStuff(); }
finally { _sem.Release(); }
```

After enough leaks, the semaphore is permanently empty — deadlocked.

### Double-release

```csharp
await _sem.WaitAsync();
try { /* ... */ }
finally {
    _sem.Release();
    _sem.Release();   // ⚠ — releases more than were acquired
}
```

Throws `SemaphoreFullException` if you exceed `maxCount`. With no maxCount, just permanently inflates the available count — silently breaks the throttle.

### Mixing across instances

```csharp
public Task M() {
    var sem = new SemaphoreSlim(1, 1);
    await sem.WaitAsync();
    // ...
}
```

Per-method semaphore is useless — no two calls share it. Make the semaphore a field of the owning class (or static for cross-instance throttling).

### Awaiting inside a `lock` that's actually a sync method

```csharp
lock (_gate) {
    await DoAsync();   // ❌ compile error
}
```

Compile error. Switch to SemaphoreSlim if you need to await inside a critical section.

### Re-entrance

Unlike `lock` (re-entrant), `SemaphoreSlim` is **NOT** re-entrant — taking it twice from the same logical flow blocks (or deadlocks):

```csharp
await _sem.WaitAsync();
try {
    await Inner();
} finally {
    _sem.Release();
}

async Task Inner() {
    await _sem.WaitAsync();   // ⚠ — deadlock; the outer await holds it
    try { /* ... */ }
    finally { _sem.Release(); }
}
```

Design to avoid recursive acquire. Or use a different primitive (AsyncLock libraries provide re-entrant async locks).

---

## Internals — how SemaphoreSlim works

`SemaphoreSlim` has:
- An `int` counter (atomic).
- A linked list / queue of waiters (each a `TaskCompletionSource<bool>`).

`Wait/WaitAsync`:
1. Atomically try to decrement the counter (if > 0, take a permit).
2. If 0, enqueue a TaskCompletionSource, return its Task (or block on its WaitHandle for sync).

`Release`:
1. If there's a waiter in the queue, dequeue them and complete their TaskCompletionSource.
2. Otherwise, increment the counter.

For waiting threads, completion happens via the thread pool (not on the releasing thread). So releasing one permit kicks off another async continuation in parallel.

### Memory cost per waiter

Each waiter holds a TaskCompletionSource + a small queue node — ~100 bytes. For thousands of pending waiters, this adds up.

---

## Async lock libraries

Several third-party libraries provide convenient async lock primitives:

- **Nito.AsyncEx** — `AsyncLock`, `AsyncSemaphore`, `AsyncMonitor`. Re-entrant variants available.
- **`Microsoft.VisualStudio.Threading`** — `AsyncSemaphore`, `AsyncReaderWriterLock`.

For most code, `SemaphoreSlim(1, 1)` is enough. Reach for libraries if you need re-entrance or specific behavior.

---

## Common patterns

### Producer-consumer throttle

```csharp
public async Task ProcessAllAsync(IEnumerable<Item> items, int parallelism, CancellationToken ct = default) {
    using var sem = new SemaphoreSlim(parallelism);
    var tasks = items.Select(async item => {
        await sem.WaitAsync(ct);
        try { await ProcessAsync(item, ct); }
        finally { sem.Release(); }
    });
    await Task.WhenAll(tasks);
}
```

Same as `Parallel.ForEachAsync` but more explicit.

### Rate limiter (with timer)

```csharp
public class TokenBucket {
    private readonly SemaphoreSlim _sem;
    public TokenBucket(int rate) {
        _sem = new(rate, rate);
        _ = Task.Run(async () => {
            while (true) {
                await Task.Delay(TimeSpan.FromSeconds(1));
                int needed = rate - _sem.CurrentCount;
                if (needed > 0) _sem.Release(needed);
            }
        });
    }
    public Task AcquireAsync(CancellationToken ct = default) => _sem.WaitAsync(ct);
}
```

Refills the bucket once per second. Quick-and-dirty rate limiting. For real systems, `Microsoft.AspNetCore.RateLimiting` is more sophisticated.

### Cancel-on-first-failure fan-out

```csharp
public async Task RunAllOrFailFastAsync<T>(IEnumerable<Func<CancellationToken, Task>> ops, CancellationToken ct = default) {
    using var sem = new SemaphoreSlim(10);
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);

    var tasks = ops.Select(async op => {
        await sem.WaitAsync(cts.Token);
        try { await op(cts.Token); }
        catch { cts.Cancel(); throw; }
        finally { sem.Release(); }
    });

    await Task.WhenAll(tasks);
}
```

If any task fails, cancel the rest. Combined with throttling.

---

## When to use

✓ Throttling concurrent async operations.
✓ Async-friendly mutual exclusion (`SemaphoreSlim(1, 1)`).
✓ Rate limiting (with help from a timer).
✓ Bounded concurrency in fan-out scenarios.

✗ Single-threaded code — overkill; just use a counter.
✗ Cross-process — use `Semaphore` (without Slim) or a kernel sync primitive.
✗ Re-entrant — SemaphoreSlim isn't re-entrant; use a library.
✗ Simple counter — `Interlocked` is lighter.

---

## Performance

- `Wait/Release` on uncontended semaphore: ~50-100 ns.
- `WaitAsync` (no contention): same — synchronous fast path.
- With contention: ~microsecond for the task continuation.
- Per-instance memory: ~hundred bytes + waiter queue.

For very-hot paths with millions of acquires per second, even SemaphoreSlim has overhead. Lock-free alternatives (Interlocked, ConcurrentBag) often win.

For typical async throttling (HTTP, DB, file): SemaphoreSlim is the right tool. Its cost is dwarfed by the I/O cost it's throttling.

→ Next: [11-Interlocked.md](11-Interlocked.md)
