# Task.WhenAll, Task.WhenAny, Task.WhenEach

> Composing multiple tasks. The most-used Task APIs after `await` itself.

---

## Task.WhenAll

Wait for **all** tasks to complete.

```csharp
Task<int> a = GetA();
Task<int> b = GetB();
Task<int> c = GetC();

int[] results = await Task.WhenAll(a, b, c);
// results[0] = a's result, results[1] = b's, etc.
```

Behavior:
- Returns a Task that completes when all input tasks complete.
- If any task throws, all are awaited first; then the result task is faulted (or canceled).
- Result type:
  - `Task.WhenAll(IEnumerable<Task>)` → returns `Task` (no values).
  - `Task.WhenAll(IEnumerable<Task<T>>)` → returns `Task<T[]>` (array of results).

For the classic "run N things concurrently and wait":

```csharp
var responses = await Task.WhenAll(urls.Select(httpClient.GetStringAsync));
```

100 HTTP calls, all in flight, ~ as long as the slowest. Vs sequential await which would be sum of all.

### Exception handling

If any task throws:

```csharp
try {
    await Task.WhenAll(taskA, taskB, taskC);
} catch (Exception ex) {
    // Catches the FIRST exception (whichever task throws first)
}
```

`await` only rethrows one exception (the first faulted task's). The **others are still in the Task's `Exception` property** as an `AggregateException`:

```csharp
var t = Task.WhenAll(taskA, taskB, taskC);
try { await t; }
catch {
    foreach (var inner in t.Exception!.InnerExceptions) {
        _log.LogError(inner, "task failed");
    }
}
```

The Task object captures all exceptions; `await` only unwraps the first. To see all, examine `t.Exception`.

### All tasks finish even on failure

```csharp
var slowFail = Task.Run(async () => { await Task.Delay(5000); throw new(); });
var slowOk = Task.Run(async () => { await Task.Delay(5000); return 42; });

try { await Task.WhenAll(slowFail, slowOk); }
catch (Exception ex) {
    // Waits the full 5 seconds — even though slowFail "failed early",
    // WhenAll waits for slowOk to finish too.
}
```

If you want **fail-fast** (cancel remaining on first failure), see the linked CTS pattern below.

### Common patterns

**Materialize multiple async calls:**
```csharp
var (user, orders) = (await GetUser(), await GetOrders());        // sequential — bad
var userTask = GetUser(); var ordersTask = GetOrders();
var (user, orders) = (await userTask, await ordersTask);          // concurrent — good
// Or:
var results = await Task.WhenAll(GetUser(), GetOrders());
```

**Fail-fast — cancel all on first failure:**
```csharp
public async Task RunAllAsync<T>(IEnumerable<Func<CancellationToken, Task<T>>> ops, CancellationToken outerCt = default) {
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(outerCt);
    var tasks = ops.Select(op => Wrap(op, cts)).ToList();
    await Task.WhenAll(tasks);

    async Task<T> Wrap(Func<CancellationToken, Task<T>> op, CancellationTokenSource cts) {
        try { return await op(cts.Token); }
        catch { cts.Cancel(); throw; }
    }
}
```

Each task observes the linked token; if any fails, it triggers cancellation of the rest.

---

## Task.WhenAny

Wait for **any one** task to complete.

```csharp
Task<int> done = await Task.WhenAny(a, b, c);
int result = await done;
```

Returns the first **completed** task — not its result. You then await that task to get the value (or its exception).

The other tasks **keep running**. You're responsible for canceling them if you don't want their results.

### Timeouts

Classic timeout pattern:

```csharp
var work = LongOperationAsync();
var timeout = Task.Delay(TimeSpan.FromSeconds(5));
var winner = await Task.WhenAny(work, timeout);
if (winner == timeout) {
    // didn't finish in time
    throw new TimeoutException();
}
int result = await work;
```

But this leaves `work` running in the background. Better: pass a CancellationToken with `CancelAfter`:

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
try { int result = await LongOperationAsync(cts.Token); }
catch (OperationCanceledException) { throw new TimeoutException(); }
```

This propagates cancellation into the operation, letting it stop cleanly.

`Task.WaitAsync(TimeSpan)` (.NET 6+) is the modern API for adding a timeout to an existing task:

```csharp
var result = await SomeAsync().WaitAsync(TimeSpan.FromSeconds(5));   // throws TimeoutException
var result = await SomeAsync().WaitAsync(cts.Token);                  // throws OperationCanceledException
```

Cleaner than WhenAny + Delay.

### Racing

```csharp
var primary = httpClient.GetStringAsync(primaryUrl);
var backup = httpClient.GetStringAsync(backupUrl);
var winner = await Task.WhenAny(primary, backup);
return await winner;
```

Whichever responds first wins. Hedged requests for latency reduction. (For real hedging, use `Microsoft.Extensions.Resilience` — handles cleanup, deduplication.)

---

## Task.WhenEach (.NET 9+)

Stream tasks **as they complete**.

```csharp
await foreach (var task in Task.WhenEach(taskA, taskB, taskC)) {
    var result = await task;
    Process(result);
}
```

Returns an `IAsyncEnumerable<Task<T>>` that yields tasks in completion order. Each iteration gives you a completed task; you await it to get the value (and handle exceptions).

Before .NET 9, you'd build this manually:

```csharp
public static async IAsyncEnumerable<Task<T>> WhenEachManual<T>(IEnumerable<Task<T>> tasks) {
    var pending = tasks.ToList();
    while (pending.Count > 0) {
        var done = await Task.WhenAny(pending);
        pending.Remove(done);
        yield return done;
    }
}
```

`Task.WhenEach` is the proper API now — efficient implementation, no O(n²) WhenAny calls.

### Use cases

- Show progress as each request finishes:
```csharp
await foreach (var t in Task.WhenEach(downloads)) {
    var data = await t;
    progressBar.Increment();
    if (data.IsImportant) break;   // can stop early
}
```

- Process results in completion order (not input order):
```csharp
await foreach (var t in Task.WhenEach(queries)) {
    try {
        await Render(await t);
    } catch (Exception ex) {
        _log.LogError(ex, "query failed");
    }
}
```

---

## Task.Wait vs await

```csharp
task.Wait();              // blocks the thread; AggregateException on failure
await task;               // releases thread; unwraps first exception

Task.WaitAll(t1, t2);     // blocks
await Task.WhenAll(t1, t2); // async equivalent

Task.WaitAny(t1, t2);     // blocks
Task<int> done = await Task.WhenAny(t1, t2);   // async equivalent
```

The `Wait*` variants are sync and block. `When*` are async and don't. In async code, always use the When* variants.

---

## Internals — how WhenAll works

```csharp
public static Task<TResult[]> WhenAll<TResult>(IEnumerable<Task<TResult>> tasks) {
    var array = tasks.ToArray();
    if (array.Length == 0) return Task.FromResult(Array.Empty<TResult>());

    var aggregate = new AggregateTask(array);
    foreach (var t in array) {
        t.ContinueWith(_ => aggregate.OnTaskComplete(), ...);
    }
    return aggregate.Task;
}
```

(Simplified.) Each input task gets a continuation. When all have called `OnTaskComplete`, the aggregate task completes — collecting results into an array (preserving order) and aggregating exceptions.

Performance:
- One Task allocation for the aggregate.
- One continuation registration per input task.
- For 1000 tasks, ~1000 small registrations + one aggregate.

Modern .NET has optimizations for common cases (small N) — the aggregate uses a lock-free completion counter instead of a heavy synchronization primitive.

### WhenAny

Same idea, but the continuation completes the aggregate **on the first** input to finish. Doesn't wait for others.

The aggregate doesn't observe later tasks' exceptions, which become **unobserved**. The runtime fires `TaskScheduler.UnobservedTaskException` for those (potentially). Use cancellation to clean up.

---

## Common patterns

### Sequential vs concurrent (the most important pattern)

```csharp
// Sequential (~3 seconds if each takes 1s):
var a = await GetA();
var b = await GetB();
var c = await GetC();

// Concurrent (~1 second):
var taskA = GetA();
var taskB = GetB();
var taskC = GetC();
var a = await taskA; var b = await taskB; var c = await taskC;

// Or shorter:
var results = await Task.WhenAll(GetA(), GetB(), GetC());
```

The pattern: **start all tasks before awaiting**. await doesn't make things concurrent — starting tasks first does.

### Heterogeneous WhenAll

`WhenAll(IEnumerable<Task>)` is non-generic — returns Task, not Task<object[]>. You await each separately:

```csharp
Task<int> ta = GetIntAsync();
Task<string> tb = GetStringAsync();
Task tc = DoSideEffectAsync();

await Task.WhenAll(ta, tb, tc);

int a = ta.Result;       // safe — task is complete
string b = tb.Result;
// tc has no result; just done
```

After WhenAll completes successfully, all tasks are complete; `.Result` is safe (and doesn't deadlock).

### Filtering tasks by result

```csharp
var responses = await Task.WhenAll(urls.Select(httpClient.GetAsync));
var ok = responses.Where(r => r.IsSuccessStatusCode);
```

### Multiple WhenAny with timeout

```csharp
// "Try fetching from cache for 100ms, then fall back to DB"
var cache = ReadFromCacheAsync();
var timeout = Task.Delay(100);
var first = await Task.WhenAny(cache, timeout);
if (first == cache) return await cache;
return await ReadFromDbAsync();   // cache too slow
```

---

## Common bugs

### Forgetting to await WhenAll

```csharp
Task.WhenAll(t1, t2);   // ⚠ — not awaited; aggregate task discarded
// Continue immediately — race condition
```

Always `await` (or `_ =` if intentionally fire-and-forget with try/catch).

### Trying to get the first result via WhenAny without re-awaiting

```csharp
var first = await Task.WhenAny(a, b);
int result = first.Result;   // ⚠ — task is complete; .Result is safe but...
// If the task is faulted, .Result throws AggregateException.
// Better:
int result = await first;     // unwraps cleanly
```

### Not handling exceptions from "losing" tasks in WhenAny

```csharp
var first = await Task.WhenAny(work, timeout);
// work might still throw later; if you don't await it, the exception is unobserved
```

Cancel the loser explicitly (via CancellationToken) so it stops cleanly. Or await it in a try/catch to consume the exception.

### Building tasks lazily in WhenAll

```csharp
await Task.WhenAll(items.Select(async i => await ProcessAsync(i)));   // ⚠ slightly weird
await Task.WhenAll(items.Select(ProcessAsync));                        // cleaner — method group
```

Both work. The method group form avoids the async lambda wrapper.

### Mixing fire-and-forget with WhenAll

```csharp
foreach (var item in items) ProcessAsync(item);   // ⚠ — fire-and-forget, return Tasks discarded
```

Should be:
```csharp
await Task.WhenAll(items.Select(ProcessAsync));
```

---

## Performance

- `Task.WhenAll(n)` allocations: 1 aggregate Task + n continuations.
- `Task.WhenAny(n)` allocations: 1 aggregate Task + n continuations (each will fire-but-ignore for losers).
- `Task.WhenEach(n)` (.NET 9): optimized internal queue, lower overhead than chained WhenAny.
- For very-large N (10,000+), consider batching to limit memory.

For typical N (100s), these are essentially free compared to the I/O work.

---

## When to use which

| Need | Use |
|---|---|
| Wait for all to finish | `Task.WhenAll` |
| Wait for any one (timeout, racing) | `Task.WhenAny` |
| Process results as they complete | `Task.WhenEach` (.NET 9+) |
| Add a timeout to a single task | `task.WaitAsync(timeout)` (.NET 6+) |
| Throttled fan-out | `Parallel.ForEachAsync` |

---

## Summary

- `Task.WhenAll` — wait for everyone; perfect for fan-out.
- `Task.WhenAny` — first one wins; useful for racing and timeouts.
- `Task.WhenEach` (.NET 9+) — stream tasks in completion order.
- `task.WaitAsync` (.NET 6+) — clean timeout per task.
- Exceptions: WhenAll collects all (await unwraps first); WhenAny ignores losers.

→ Next: [16-MemoryModelVolatile.md](16-MemoryModelVolatile.md)
