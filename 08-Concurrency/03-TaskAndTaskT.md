# Task and Task&lt;T&gt;

## What it is

`Task` represents an **asynchronous operation** — a promise of future completion. `Task<T>` is one that produces a value of type T. They're the fundamental currency of async C#.

```csharp
Task t = SomethingAsync();           // an operation that completes (no value)
Task<int> tv = SomethingAsync();      // an operation that completes with an int

await t;
int v = await tv;
```

Tasks are heap-allocated objects. They have state (Created, Running, Completed, Faulted, Canceled), a result (for `Task<T>`), and possibly an exception. You build them implicitly via `async` methods, or explicitly via `Task.Run`, `Task.FromResult`, `TaskCompletionSource`, etc.

---

## Why they exist

Pre-Task .NET had several async patterns (EAP — Event-based Asynchronous Pattern; APM — Asynchronous Programming Model with Begin/End). None were composable. Task became the **single unified primitive** for "something that finishes later," enabling `async/await` to be syntactic sugar over a clean object model.

Modern code: you almost never construct Tasks manually — `async` methods produce them. But you need to understand them to debug, compose, and pass them around.

---

## Task lifecycle

A Task moves through states:

```
Created → WaitingForActivation → WaitingToRun → Running → RanToCompletion
                                                      → Faulted
                                                      → Canceled
```

Most awaited tasks start in `WaitingForActivation` (returned from an async method) — already running or about to be. `Created` is uncommon (you'd have to use `Task` constructor + `Start`, almost never done).

Inspect state:
```csharp
Task t = SomeAsync();
t.Status;            // TaskStatus enum
t.IsCompleted;        // true for completion + faulted + canceled
t.IsCompletedSuccessfully;  // .NET 5+ — true only for clean completion
t.IsFaulted;          // exception thrown
t.IsCanceled;         // canceled via CancellationToken
```

Most async code uses `await`, which throws on faulted/canceled. The state inspection APIs are for advanced cases (custom schedulers, library code).

---

## Producing Tasks

### From an async method

```csharp
public async Task<int> GetCountAsync() {
    await Task.Delay(100);
    return 42;
}

Task<int> task = GetCountAsync();   // starts immediately; returns task
```

99% of how you'll get tasks. The async method's state machine creates and manages the Task.

### Task.Run — CPU work on the thread pool

```csharp
Task<int> task = Task.Run(() => HeavyCompute());
Task<int> task2 = Task.Run(async () => {
    await Task.Delay(100);
    return ComputeMore();
});
```

For CPU-bound or sync-only code that you want to run on the pool. **Don't** wrap async I/O in Task.Run — it's wasteful (see [§01](01-ThreadsVsTasks.md)).

### Task.FromResult — already-completed task

```csharp
public Task<int> GetCached() => Task.FromResult(42);

public async Task<int> Cached(int key) {
    if (_cache.TryGetValue(key, out var value)) return value;   // wraps in Task<int>
    return await LoadFromDb(key);
}
```

`Task.FromResult(value)` returns a completed `Task<T>`. No allocation deep enough to matter; the runtime sometimes caches common ones (e.g., `Task.FromResult(true)`).

For the **completed-synchronously** path of a method, this is the idiom. `await` on a `Task.FromResult` is essentially free.

### Task.CompletedTask

```csharp
public Task DoSomething() {
    if (alreadyDone) return Task.CompletedTask;
    return ActuallyDoSomethingAsync();
}
```

A pre-allocated completed `Task` (no value). Singleton — no allocation per call.

### Task.FromException — failed task

```csharp
public Task<int> Fail(Exception ex) => Task.FromException<int>(ex);
```

A task that's already faulted with the given exception. Awaiting it rethrows.

### Task.FromCanceled

```csharp
public Task<int> Cancel(CancellationToken ct) => Task.FromCanceled<int>(ct);
```

A task that's in the Canceled state. Awaiting it throws `OperationCanceledException`.

### TaskCompletionSource — manual completion

For bridging non-async code (events, callbacks, raw I/O) to a Task:

```csharp
public Task<int> BridgeToCallback() {
    var tcs = new TaskCompletionSource<int>();
    SomeOldApi.OnComplete(result => tcs.SetResult(result));
    SomeOldApi.OnError(ex => tcs.SetException(ex));
    return tcs.Task;
}
```

You hand out `tcs.Task` to callers. Then code somewhere else calls `tcs.SetResult` / `SetException` / `SetCanceled`.

Used to wrap event-based APIs, timer callbacks, native interop. Less commonly needed since most APIs are async-native now.

**Always** use `TaskCreationOptions.RunContinuationsAsynchronously` to avoid running continuations on the thread that calls `SetResult`:

```csharp
var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);
```

Without it, a continuation might re-enter your code on the wrong thread or hold the wrong lock.

---

## Awaiting

```csharp
Task<int> task = GetAsync();
int v = await task;
```

`await`:
- If the task is complete, gets the result (or rethrows the exception).
- If running, registers a continuation and returns.

Awaiting a faulted task **rethrows** the original exception:
```csharp
try {
    await task;
} catch (HttpRequestException ex) {
    // original exception, not wrapped
}
```

Awaiting a canceled task throws `OperationCanceledException` (or a subclass).

---

## Task.Wait, Task.Result — blocking

```csharp
task.Wait();           // block until complete
int v = task.Result;   // block + get result
```

**Avoid these** in async code. They block the calling thread. They also wrap exceptions in `AggregateException`:

```csharp
try {
    int v = task.Result;
} catch (AggregateException ex) {
    var inner = ex.InnerException;   // dig out the real exception
}
```

`await` is the right tool 99% of the time. `.Result` is for top-level sync code where you have no choice (e.g., a constructor that needs an async result — design around this if you can).

---

## ContinueWith — the old continuation API

Pre-async/await, you'd compose tasks with `ContinueWith`:

```csharp
GetAsync().ContinueWith(t => {
    if (t.IsFaulted) { ... }
    else { var v = t.Result; ... }
});
```

Still works, but **prefer async/await**. It's cleaner and has correct exception handling.

`ContinueWith` is useful for:
- Side-effect continuations that must always run (logging, cleanup) — though `await` + try/finally is usually cleaner.
- Custom TaskScheduler scenarios — rare.

---

## Task.Delay — async sleep

```csharp
await Task.Delay(1000);                     // 1 second
await Task.Delay(TimeSpan.FromSeconds(1));
await Task.Delay(1000, cancellationToken);  // cancellable
```

Internally uses a Timer. Doesn't block a thread. Wakes up an async continuation when the timer fires.

For sync code, `Thread.Sleep` is the equivalent — but blocks the thread.

```csharp
Thread.Sleep(1000);    // BAD in async code — wastes a thread
await Task.Delay(1000); // GOOD — releases the thread
```

---

## Task composition

### Task.WhenAll

Run multiple tasks concurrently; wait for all to finish:

```csharp
Task<int> a = GetA();
Task<int> b = GetB();
Task<int> c = GetC();
int[] results = await Task.WhenAll(a, b, c);
```

Returns when all complete. If any throws, exception propagates **after** all finish.

### Task.WhenAny

Wait for the first to complete:

```csharp
Task<int> first = await Task.WhenAny(taskA, taskB);
```

Returns the first **completed** task (not its result — you await it again). Used for racing, timeouts:

```csharp
var fetch = httpClient.GetAsync(url);
var timeout = Task.Delay(5000);
var winner = await Task.WhenAny(fetch, timeout);
if (winner == timeout) throw new TimeoutException();
var response = await fetch;
```

For built-in timeouts, prefer `cancellationToken.CancelAfter(timeout)` and let the original task handle cancellation cleanly.

### Task.WhenEach (.NET 9+)

Process tasks **as they complete**, not after all:

```csharp
await foreach (var task in Task.WhenEach(tasks)) {
    var result = await task;
    Process(result);
}
```

Useful for streaming results from a fan-out — show partial progress as each request finishes.

[§15 WhenAllWhenAny](15-TaskWhenAllWhenAny.md) goes deep on composition.

---

## Internals — what a Task object holds

Approximate Task layout:

```csharp
public class Task {
    private int m_stateFlags;             // status + flags
    private object? m_continuationObject;  // single continuation or list
    private object? m_taskScheduler;
    private object? m_action;               // delegate (for Task.Run)
    private object? m_stateObject;          // optional state
    private CancellationToken m_cancellationToken;
    private ManualResetEventSlim? m_completionEvent;   // for Wait()
    // ...
}
```

For `Task<T>`, there's also `m_result`.

Per-Task overhead is roughly 100-200 bytes. Creating Tasks has been heavily optimized in modern .NET — typical Task creation is ~50-100 ns.

For tasks that always complete synchronously (cached values), prefer `Task.FromResult` (no allocation in some cases) or `ValueTask<T>` (no allocation at all).

### Task.FromResult caching

For some common values, the runtime caches:

```csharp
Task.FromResult(true);    // singleton — no allocation
Task.FromResult(false);
Task.FromResult(0);       // sometimes cached
Task.CompletedTask;       // singleton
```

Don't rely on identity (`ReferenceEquals`) — caching is an implementation detail. But know that hot paths returning `true` / `false` Tasks are essentially free.

---

## Where to use what

| Need | Use |
|---|---|
| Async method, has a result | `async Task<T>` returning T |
| Async method, no result | `async Task` |
| Hot-path async with cached values | `async ValueTask<T>` |
| Bridge a callback to async | `TaskCompletionSource<T>` |
| Run sync code asynchronously | `Task.Run(() => sync())` |
| Already-known result | `Task.FromResult(value)` |
| Already-finished void task | `Task.CompletedTask` |
| Already-failed task | `Task.FromException<T>(ex)` |

---

## Common bugs

- **Returning `null` for `Task`** — `null` is a valid Task return technically, but awaiting it throws NullReferenceException. Return `Task.CompletedTask` or `Task.FromResult(default(T))`.
- **Forgetting to await** — `Task t = SomeAsync();` starts the work but doesn't wait. If you didn't intend fire-and-forget, you have a bug.
- **Awaiting twice** — `Task<T>` can be awaited multiple times (gives the same result/exception each time). `ValueTask<T>` cannot.
- **Capturing context unintentionally** — see [§06 ConfigureAwait](06-ConfigureAwait.md).
- **Wrapping sync code in `Task.Run` for "async"** — the call is async-shaped but the work still blocks a thread. Better: make the underlying work truly async.

---

## Performance

- `Task<T>` allocation: ~100-200 bytes.
- `Task.FromResult(value)` for common values: cached, zero allocation.
- `await` on already-complete task: synchronous fast path, near-free.
- `await` on running task: state machine box (~100 bytes) + continuation registration.
- Modern .NET optimizations (.NET 8+, .NET 10 escape analysis) reduce allocations dramatically.

For hot paths where Tasks dominate, see ValueTask in [§04](04-ValueTask.md).

→ Next: [04-ValueTask.md](04-ValueTask.md)
