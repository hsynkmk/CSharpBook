# Common Async Bugs

> The 12 most common async-related bugs in C#, what they look like, why they happen, and how to fix them. If you've shipped async code, you've probably written some of these.

---

## 1. `.Result` / `.Wait()` causing deadlock

```csharp
// UI thread (WPF / WinForms)
var data = SomeAsync().Result;   // ⚠ deadlock!

public async Task<string> SomeAsync() {
    await Task.Delay(100);
    return "...";
}
```

The flow:
1. `.Result` blocks the UI thread.
2. `Task.Delay(100)` completes.
3. The continuation (after await) needs to post back to the UI thread.
4. UI thread is blocked waiting for the Task.
5. **Deadlock.**

**Fix**: never block on async. `async` all the way:
```csharp
var data = await SomeAsync();
```

If you absolutely must (e.g., in a constructor), use `ConfigureAwait(false)` in the library OR rethink the design.

ASP.NET Core has no SynchronizationContext, so `.Result` doesn't deadlock there — but still wastes a thread.

---

## 2. `async void` outside event handlers

```csharp
public async void DoIt() {
    await Task.Delay(100);
    throw new InvalidOperationException("oops");
}

DoIt();   // exception is unobservable!
```

`async void` exceptions:
- Can't be awaited.
- Propagate to the SynchronizationContext (UI: bubble up, ASP.NET Core: crash).
- Caller has no way to know when the method finishes.

**Fix**: return `Task` for non-event-handlers:
```csharp
public async Task DoIt() { ... }
await DoIt();
```

The only legal `async void` is for event handlers (the signature requires `void`). Even then, wrap in try/catch:

```csharp
public async void OnClick(object sender, EventArgs e) {
    try {
        await DoWorkAsync();
    } catch (Exception ex) {
        _log.LogError(ex, "click failed");
    }
}
```

---

## 3. Sync-over-async with custom code

```csharp
public string LoadConfig() {
    return SomeAsync().GetAwaiter().GetResult();   // ⚠
}
```

`GetAwaiter().GetResult()` is the "official" way to block on async, with unwrapped exceptions (vs `.Result` which wraps in AggregateException). Same dangers: deadlock, thread waste.

**Fix**: make the caller async too. The async chain is contagious; let it propagate:

```csharp
public async Task<string> LoadConfigAsync() {
    return await SomeAsync();
}
```

If the caller is a top-level entry point that can't be async, OK — call once at the top:
```csharp
static int Main() {
    return RunAsync().GetAwaiter().GetResult();
}
```

But anywhere in the middle, async-all-the-way.

---

## 4. Async-over-sync (wrapping sync in `Task.Run`)

```csharp
public Task<int> GetAsync() {
    return Task.Run(() => SyncMethod());   // ⚠ fake async
}
```

Looks async, isn't really. The Task.Run grabs a thread pool thread, runs the sync method, blocks the thread. Caller awaits, fine — but you've burned a thread pool slot for nothing.

**Fix**: make `SyncMethod` truly async, or accept that it's sync. Don't wrap.

OK use case: offloading sync CPU work in a UI app to keep the UI thread responsive. There the thread cost is acceptable.

---

## 5. Forgetting to await

```csharp
public async Task M() {
    SomeAsync();   // ⚠ — Task returned but ignored
    Console.WriteLine("done");
}
```

`SomeAsync()` starts. `M` continues immediately. Console line prints before SomeAsync finishes. If SomeAsync throws, the exception is **unobserved** — silently swallowed in modern .NET (logged via `TaskScheduler.UnobservedTaskException` if you've hooked it).

Compiler warning: **CS4014** ("Because this call is not awaited, execution of the current method continues before the call is completed").

**Fix**:
```csharp
await SomeAsync();
```

If you genuinely want fire-and-forget, use `_ =`:
```csharp
_ = SomeAsync().ContinueWith(t => _log.LogError(t.Exception, "fire-and-forget failed"),
    TaskContinuationOptions.OnlyOnFaulted);
```

Or wrap in `Task.Run` with try/catch. Be deliberate.

---

## 6. Awaiting a Task<T> twice (with ValueTask)

```csharp
ValueTask<int> vt = GetAsync();
int a = await vt;
int b = await vt;   // ⚠ undefined behavior
```

`ValueTask` is one-shot. Awaiting twice may return stale data, throw, or work — non-deterministic.

For `Task<T>`, awaiting multiple times is fine — same result each time.

**Fix**: capture the result:
```csharp
int v = await GetAsync();
int a = v; int b = v;
```

Or convert to Task:
```csharp
Task<int> t = vt.AsTask();
await t; await t;   // OK
```

---

## 7. Captured loop variable in async lambda

```csharp
for (int i = 0; i < 10; i++) {
    _ = Task.Run(async () => {
        await Task.Delay(100);
        Console.WriteLine(i);   // ⚠ all see final value of i
    });
}
```

The classic closure bug — async makes it more subtle because the printouts happen later.

**Fix**: copy the variable per iteration, or use `foreach`:
```csharp
for (int i = 0; i < 10; i++) {
    int copy = i;
    _ = Task.Run(async () => { await Task.Delay(100); Console.WriteLine(copy); });
}

// Or
foreach (var i in Enumerable.Range(0, 10)) {
    _ = Task.Run(async () => { await Task.Delay(100); Console.WriteLine(i); });
}
```

`foreach` captures a fresh variable per iteration.

---

## 8. Not forwarding CancellationToken

```csharp
public async Task ProcessAsync(CancellationToken ct) {
    await DoSlowThing();          // ⚠ doesn't see ct
    await AnotherSlow();
}
```

The method accepts a token but doesn't forward it. Inner async ops ignore cancellation. Caller cancels → nothing happens; the method runs to completion.

**Fix**: forward the token to every inner async call:
```csharp
public async Task ProcessAsync(CancellationToken ct) {
    await DoSlowThing(ct);
    await AnotherSlow(ct);
    ct.ThrowIfCancellationRequested();
    DoSyncWork();
}
```

---

## 9. Lock inside async

```csharp
public async Task M() {
    lock (_gate) {
        await Task.Delay(100);   // ❌ compile error
    }
}
```

Compiler catches this — `lock` can't span `await`. The reason: `Monitor.Enter` ties the lock to the OS thread; after `await`, you might be on a different thread.

**Fix**: use `SemaphoreSlim`:
```csharp
private readonly SemaphoreSlim _sem = new(1, 1);
public async Task M() {
    await _sem.WaitAsync();
    try {
        await Task.Delay(100);   // OK
    } finally {
        _sem.Release();
    }
}
```

---

## 10. ConfigureAwait(false) confusion

```csharp
// In a WPF library
public async Task<string> DownloadAsync() {
    var data = await _http.GetStringAsync(url);   // ⚠ captures UI context
    return data;
}
```

If the caller blocks via `.Result`, deadlock.

**Fix in libraries**: `ConfigureAwait(false)` everywhere:
```csharp
public async Task<string> DownloadAsync() {
    var data = await _http.GetStringAsync(url).ConfigureAwait(false);
    return data;
}
```

For ASP.NET Core app code: no SynchronizationContext, ConfigureAwait does nothing — skip it.

Don't add `ConfigureAwait(false)` to UI app code if you want the continuation back on the UI thread (typical for button handlers).

See [§06 ConfigureAwait](06-ConfigureAwait.md) for the full story.

---

## 11. Concurrent Task.Run on shared state without sync

```csharp
private List<int> _results = new();

public async Task ProcessAllAsync(IEnumerable<Item> items) {
    var tasks = items.Select(item => Task.Run(() => {
        var r = Process(item);
        _results.Add(r);   // ⚠ List<T> isn't thread-safe
    }));
    await Task.WhenAll(tasks);
}
```

Concurrent Adds to `List<T>` can corrupt the internal state. Race condition.

**Fix**: use a concurrent collection or accumulate locally:
```csharp
var bag = new ConcurrentBag<int>();
var tasks = items.Select(item => Task.Run(() => bag.Add(Process(item))));
await Task.WhenAll(tasks);
_results = bag.ToList();
```

Or `Parallel.ForEachAsync` with per-thread state.

---

## 12. Disposing async resources synchronously

```csharp
using var ctx = new AppDbContext();
await ctx.Users.ToListAsync();
// At end of using: ctx.Dispose() — blocks while flushing pending writes
```

The using statement calls `Dispose()` (sync). For DbContext (and Stream, HttpClient, etc.), there's a better `DisposeAsync` that does the cleanup async.

**Fix**: `await using`:
```csharp
await using var ctx = new AppDbContext();
await ctx.Users.ToListAsync();
// At end: await ctx.DisposeAsync()
```

See [§08 AsyncDisposable](08-AsyncDisposable.md).

---

## Honorable mentions

### Misusing `IAsyncEnumerable<T>`

```csharp
public async IAsyncEnumerable<int> Produce(CancellationToken ct) {   // ⚠ no [EnumeratorCancellation]
    for (int i = 0; i < 100; i++) {
        await Task.Delay(100, ct);
        yield return i;
    }
}

// Consumer
await foreach (var x in producer.WithCancellation(myCts.Token)) { ... }
```

Without `[EnumeratorCancellation]`, the consumer's token doesn't reach the iterator's awaits.

```csharp
public async IAsyncEnumerable<int> Produce(
    [EnumeratorCancellation] CancellationToken ct = default)   // ✓
```

### Multiple enumeration of an async stream

```csharp
var stream = ProduceAsync();
await foreach (var x in stream) { ... }
await foreach (var x in stream) { ... }   // ⚠ — iterator is single-pass
```

Materialize once if you need multi-iteration:
```csharp
var list = await stream.ToListAsync();
```

### `Task.Run(() => async lambda)` and the missing await

```csharp
Task.Run(async () => {
    await SomeAsync();
});   // ⚠ Task returned ignored
```

Should be `_ = Task.Run(...)` or `await Task.Run(...)`. Pattern depends on intent.

### Awaiting in a hot loop

```csharp
for (int i = 0; i < 1000; i++) {
    await ProcessAsync(items[i]);   // sequential — 1000x slower than concurrent
}
```

If items are independent, parallelize:
```csharp
await Task.WhenAll(items.Select(ProcessAsync));
// Or with throttling:
await Parallel.ForEachAsync(items, async (item, ct) => await ProcessAsync(item));
```

### Forgetting Task.Yield in a busy loop

```csharp
public async Task TightLoop() {
    while (true) {
        // CPU work
        // No await — locks the thread forever
    }
}
```

Yield periodically:
```csharp
public async Task TightLoop() {
    while (true) {
        // CPU work
        await Task.Yield();   // give other work a chance
    }
}
```

Or use `Task.Run` to put it on a dedicated pool thread.

---

## The mental checklist

When writing async code:

- [ ] Method returns `Task` / `Task<T>` / `ValueTask<T>` (not `void`, except event handlers).
- [ ] All inner async calls are `await`ed.
- [ ] `CancellationToken` parameter forwarded everywhere.
- [ ] `ConfigureAwait(false)` in library code; default in app code.
- [ ] No `.Result` / `.Wait()` / `.GetAwaiter().GetResult()` in async code.
- [ ] No `lock` around `await`. Use `SemaphoreSlim` for async-safe mutex.
- [ ] Shared state mutated via concurrent collections, Interlocked, or proper locking.
- [ ] Long-running sync work offloaded with `Task.Run` (only for CPU-bound).
- [ ] Async streams use `[EnumeratorCancellation]`.
- [ ] `await using` for `IAsyncDisposable`.
- [ ] Fire-and-forget tasks have try/catch + logging.

---

## How to find these bugs

- **Roslyn analyzers** — `Microsoft.VisualStudio.Threading.Analyzers` catches most async anti-patterns (VS-prefixed rules).
- **CA2007** — `ConfigureAwait` missing in library code.
- **CS4014** — Task return value not awaited.
- **VSTHRD108** — Async-over-sync patterns.
- **VSTHRD110** — Observe result of async calls.
- Code review: look for `.Result`, `.Wait()`, `async void` outside event handlers.

For production: **structured logging of unobserved task exceptions**:

```csharp
TaskScheduler.UnobservedTaskException += (sender, e) => {
    _log.LogError(e.Exception, "unobserved task exception");
    e.SetObserved();   // prevent process crash on older runtimes
};
```

---

## Summary

The 12 most common async bugs:
1. `.Result` / `.Wait()` deadlocks.
2. `async void` outside event handlers.
3. Sync-over-async via `GetAwaiter().GetResult()`.
4. Async-over-sync (fake async via `Task.Run`).
5. Forgetting to await.
6. Double-awaiting `ValueTask<T>`.
7. Loop-variable capture in async lambdas.
8. Not forwarding CancellationToken.
9. `lock` around `await` (compile error, but related Semaphore mistakes).
10. ConfigureAwait confusion (over- or under-use).
11. Concurrent mutation of shared state without sync.
12. Sync Dispose of async-disposable resources.

Avoid these and you've sidestepped most async pain.

→ Continue: [Questions.md](Questions.md)
