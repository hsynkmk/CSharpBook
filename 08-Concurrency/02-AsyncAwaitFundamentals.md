# async / await — Fundamentals

## What it is

`async` and `await` are C# keywords that let you write asynchronous code as if it were sequential. The compiler rewrites your method into a **state machine** that pauses at each `await` and resumes when the awaited operation completes — **without blocking a thread**.

```csharp
public async Task<string> FetchAsync(string url) {
    using var client = new HttpClient();
    var response = await client.GetStringAsync(url);   // ← pause here, free the thread
    return response.ToUpper();                          // ← resume here when response arrives
}
```

While `GetStringAsync` is waiting for the network, the calling thread is freed to do other work. When the response arrives, the runtime schedules the continuation (the code after `await`) on a thread.

**Result**: thousands of concurrent network operations on a handful of threads. This is the entire reason modern .NET can handle high-throughput web servers without a thread-per-request model.

---

## Why it exists

Before async/await (.NET 4.5, 2012), async I/O looked like this:

```csharp
// Callback hell
client.BeginGetString(url, (ar) => {
    var response = client.EndGetString(ar);
    response = response.ToUpper();
    callback(response);
}, null);
```

Or:

```csharp
// Task with .ContinueWith
client.GetStringAsync(url).ContinueWith(t => {
    var response = t.Result.ToUpper();
    callback(response);
});
```

Both are unreadable for non-trivial logic. async/await collapses callback chains into linear code:

```csharp
var response = await client.GetStringAsync(url);
return response.ToUpper();
```

Same machinery underneath; vastly better syntax.

---

## The basics

### An async method

```csharp
public async Task<int> Square(int x) {
    await Task.Delay(100);
    return x * x;
}
```

Properties:
- Marked `async`.
- Returns `Task`, `Task<T>`, `ValueTask`, `ValueTask<T>`, or `void` (event handlers only).
- Body can use `await`.

The `async` keyword is **purely a compiler instruction** — it doesn't change the method's signature or behavior externally. From a caller's perspective, the method returns `Task<int>` like any other.

### `await`

```csharp
int result = await Square(5);
```

`await` does three things:
1. If the task is **already complete**, fetch the result immediately. No suspension.
2. Otherwise, **register a continuation** with the task and return.
3. When the task completes, the continuation runs (possibly on a different thread).

After `await`, the next line runs with `result` set to the task's value.

### Returning a value

```csharp
public async Task<int> Get() {
    await Task.Delay(100);
    return 42;
}

int v = await Get();
```

The compiler wraps the return value in `Task<int>`. The caller awaits the task and gets the unwrapped value.

### Returning nothing (Task)

```csharp
public async Task Process() {
    await Task.Delay(100);
}

await Process();   // no value to assign
```

`Task` is the async equivalent of `void`. Awaiting it just waits for completion.

### Returning truly nothing (void)

```csharp
public async void Click(object sender, EventArgs e) {
    await DoWorkAsync();
}
```

`async void` is **only for event handlers**. Otherwise it's an anti-pattern:
- The caller can't await it.
- Exceptions propagate to the SynchronizationContext (or crash the process).
- No way to know when it finishes.

[§17 CommonAsyncBugs](17-CommonAsyncBugs.md) covers the dangers.

---

## How execution flows

Trace this code:

```csharp
public async Task<string> Top() {
    Console.WriteLine("1");
    var task = Inner();
    Console.WriteLine("2");
    string r = await task;
    Console.WriteLine($"3 — got '{r}'");
    return r;
}

public async Task<string> Inner() {
    Console.WriteLine("A");
    await Task.Delay(100);
    Console.WriteLine("B");
    return "hello";
}

await Top();
```

Output:
```
1
A
2
B
3 — got 'hello'
```

What happens:
1. `Top` runs. Prints "1".
2. Calls `Inner()`. Inner runs synchronously up to its first await — prints "A", hits `Task.Delay`. Returns the incomplete Task.
3. Top has the Task but doesn't await yet — prints "2".
4. Top hits `await task`. The task isn't complete yet. Top suspends.
5. ~100 ms later, `Task.Delay` completes.
6. Inner resumes — prints "B", returns "hello".
7. Top's continuation fires — prints "3 — got 'hello'", returns "hello".

The key insight: **async methods run synchronously up to the first incomplete await**. Then they suspend. Then they resume from where they left off — possibly on a different thread.

---

## What `await` doesn't do

- **`await` doesn't create a thread.** It registers a continuation with an existing task and returns. No threads are summoned.
- **`await` doesn't make code parallel.** It just doesn't block. To parallelize, START multiple tasks before awaiting them:

```csharp
// Sequential (~300ms total):
var a = await Get();
var b = await Get();
var c = await Get();

// Concurrent (~100ms total):
var ta = Get();
var tb = Get();
var tc = Get();
await Task.WhenAll(ta, tb, tc);
var a = ta.Result; var b = tb.Result; var c = tc.Result;

// Or shorter:
var results = await Task.WhenAll(Get(), Get(), Get());
```

The trick is to start all tasks first, then await. Each starts immediately; the await just waits for completion.

---

## Exceptions

Exceptions propagate naturally through `await`:

```csharp
public async Task<int> Risky() {
    await Task.Delay(100);
    throw new InvalidOperationException("oops");
}

try {
    int v = await Risky();
} catch (InvalidOperationException ex) {
    Console.WriteLine(ex.Message);   // "oops"
}
```

The exception is stored in the Task. `await` re-throws it. Looks just like synchronous try/catch.

For aggregate exceptions (`Task.WhenAll`), only the **first** exception is rethrown by `await` — the others are still on the Task's `Exception` property. See [§15 WhenAllWhenAny](15-TaskWhenAllWhenAny.md).

---

## Don't block on async

```csharp
// 🚨 anti-patterns:
var result = SomeAsync().Result;
var result = SomeAsync().GetAwaiter().GetResult();
SomeAsync().Wait();
Task.WaitAll(asyncTasks);
```

These BLOCK the calling thread until the task completes. Three problems:
1. Wastes a thread.
2. If on a thread with SynchronizationContext (UI), **deadlock** — the continuation needs the UI thread; the UI thread is blocked waiting.
3. Exceptions are wrapped in `AggregateException`.

**Always async all the way down.** If a method's caller is async, the method should be too — bubble it up.

If you absolutely must block (e.g., `static void Main` in an old console app pre-C# 7.1), use:
```csharp
SomeAsync().GetAwaiter().GetResult();
```
And only at the very top level. Never in library code.

Modern code: `static async Task Main()`.

---

## What the compiler generates

For:
```csharp
public async Task<int> Square(int x) {
    await Task.Delay(100);
    return x * x;
}
```

The compiler generates roughly:
```csharp
public Task<int> Square(int x) {
    var stateMachine = new <Square>d__0 {
        x = x,
        builder = AsyncTaskMethodBuilder<int>.Create(),
        state = -1
    };
    stateMachine.builder.Start(ref stateMachine);
    return stateMachine.builder.Task;
}

private struct <Square>d__0 : IAsyncStateMachine {
    public int x;
    public int state;
    public AsyncTaskMethodBuilder<int> builder;
    private TaskAwaiter awaiter;

    public void MoveNext() {
        int result;
        try {
            if (state == -1) {
                awaiter = Task.Delay(100).GetAwaiter();
                if (!awaiter.IsCompleted) {
                    state = 0;
                    builder.AwaitUnsafeOnCompleted(ref awaiter, ref this);
                    return;
                }
            } else {
                state = -1;
            }
            awaiter.GetResult();
            result = x * x;
        } catch (Exception ex) {
            state = -2;
            builder.SetException(ex);
            return;
        }
        state = -2;
        builder.SetResult(result);
    }

    public void SetStateMachine(IAsyncStateMachine s) { /* used by hot-path optimization */ }
}
```

Key observations:
- The state machine is a **struct** (modern .NET; was a class pre-CLR optimization).
- `MoveNext` is called once initially (synchronously, by `Start`).
- If it suspends, the state field tracks where to resume.
- The `AsyncTaskMethodBuilder<T>` produces and finishes the Task.
- The continuation is queued via `AwaitUnsafeOnCompleted` — the task's completion will call `MoveNext` again with the new state.

When the awaited task completes, MoveNext is called by the task's continuation hook. It picks up at the saved state, gets the result, runs your code after the await, and (eventually) calls `SetResult` on the builder.

### Async state machine boxing

If the method has any `await`, the state machine struct gets **boxed** to the heap on first suspension (it needs a stable identity for the continuation). For methods that always complete synchronously, no boxing — pure performance.

.NET 8+ has further optimization: the runtime caches a single "boxed state machine" allocation per call frame, reusing it if possible (the `[AsyncMethodBuilder]` machinery).

.NET 10 added "box elision" for common patterns — when the runtime can prove the state machine doesn't escape, it avoids the box entirely.

---

## await on completed tasks — synchronous fast path

If the awaited task is **already complete**, `await` runs the rest of the method **synchronously**:

```csharp
public Task<int> CachedAsync() => Task.FromResult(42);

async Task M() {
    int v = await CachedAsync();   // runs synchronously — no suspension
    // ...
}
```

The state machine notices the task is complete (`awaiter.IsCompleted == true`), skips the suspension, gets the result, runs the continuation right there.

This makes "cache hit returns immediately" patterns fast. Combined with `ValueTask<T>` (next file), it lets you skip the Task allocation entirely for the common case.

---

## Where async/await shines

- **HTTP servers** — thousands of concurrent requests on a few threads.
- **HTTP clients** — fan out many requests.
- **Database access** — `ToListAsync`, `SaveChangesAsync` (EF Core).
- **File I/O** — `File.ReadAllTextAsync`, streaming reads.
- **UI** — keep the UI thread responsive during long operations.

It's the default for any I/O-bound code today.

---

## Where it doesn't help

- **CPU-bound work** — async/await doesn't make code faster. For CPU-bound, use `Task.Run` to offload to the thread pool. `Parallel.For` or PLINQ for data parallelism.
- **Single-threaded work that doesn't yield** — pure computation. Don't add async ceremony.
- **Trivial methods** — `async Task<int> Get() => 5;` has overhead. If no await, just return the value.

---

## Patterns to know

### Async all the way

```csharp
public async Task<User> GetUserAsync(int id) {
    var user = await _db.Users.FindAsync(id);
    user.LastSeen = DateTime.UtcNow;
    await _db.SaveChangesAsync();
    return user;
}
```

Every level of the call stack is async. No `.Result`, no `.Wait()`.

### Fire-and-forget (when you really mean it)

```csharp
_ = Task.Run(async () => {
    try { await LongRunning(); }
    catch (Exception ex) { _log.LogError(ex, "background failed"); }
});
```

The `_ =` makes the compiler shut up about the unawaited task. The try/catch is critical — unhandled exceptions in fire-and-forget tasks get logged via `TaskScheduler.UnobservedTaskException` (if anyone listens), or are silently dropped in modern .NET.

For truly long-running background work, use `IHostedService` / `BackgroundService` instead. See [DotNetBook Chapter 03](../../DotNetBook/03-HostingAndDI/README.md).

### Parallelization via Task.WhenAll

```csharp
var responses = await Task.WhenAll(
    httpClient.GetStringAsync(url1),
    httpClient.GetStringAsync(url2),
    httpClient.GetStringAsync(url3)
);
```

All three requests start immediately. `WhenAll` returns when all are done. Result is `string[]` matching the input order.

### Async lambdas

```csharp
items.Select(async item => await ProcessAsync(item));   // IEnumerable<Task<T>>
await Task.WhenAll(items.Select(async item => await ProcessAsync(item)));
```

Lambdas can be `async`. The selector's return type becomes Task<T>.

---

## Common bugs (preview)

- **`async void`** anywhere except event handlers — exceptions vanish.
- **`.Result` / `.Wait()`** — blocks; potential deadlock; wastes a thread.
- **Forgetting to await** — `DoWorkAsync();` (no await) starts work but doesn't wait. Possibly fire-and-forget — fine if intentional, broken if not.
- **Async-over-sync** — wrapping sync I/O in `Task.Run` to make it "async". Adds overhead without making it truly async.
- **Sync-over-async** — calling `.Result` on an async method from sync code. Blocks and risks deadlock.

[§17](17-CommonAsyncBugs.md) covers all these.

---

## When to use async/await

✓ I/O-bound work (network, database, file).
✓ Calls to other async APIs.
✓ UI code that would otherwise freeze.
✓ Anything that might wait for something.

✗ Pure CPU work — use Task.Run / Parallel instead.
✗ Trivial getters returning constants — just return the value.
✗ Sync wrappers around sync code — adds complexity for no benefit.

---

## Performance summary

- The first `await` of an incomplete task: ~few hundred nanoseconds (state machine box, continuation setup).
- Subsequent `await`s (already complete): nearly free.
- An `async Task<T>` method that suspends: ~one allocation for the state machine box + Task.
- An `async ValueTask<T>` method that completes synchronously: ZERO allocations.
- The thread pool handles fan-out efficiently — async I/O on thousands of operations uses only a few threads.

`async/await` is essentially **free** for I/O-bound work. For CPU-bound, the overhead is real but small; profile if it's hot.

→ Next: [03-TaskAndTaskT.md](03-TaskAndTaskT.md)
