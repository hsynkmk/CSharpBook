# Threads vs Tasks

## What it is

Two ways to do work concurrently:

- **`Thread`** — an OS-level construct. The runtime asks the kernel for a new thread (a stack, a scheduler entry, kernel bookkeeping). Heavyweight. ~1 MB stack each. Slow to create.
- **`Task`** — a unit of work scheduled on the **thread pool**. The pool is a small fixed pool of worker threads (typically equal to logical CPU count + some growth). Lightweight — a Task is just a heap object representing future work.

```csharp
// Thread — OS thread, dedicated stack, kernel-scheduled
var t = new Thread(() => Console.WriteLine("from a thread"));
t.Start();
t.Join();

// Task — runs on the thread pool
var task = Task.Run(() => Console.WriteLine("from a task"));
await task;
```

For 99% of concurrency in modern C# you use `Task`. `Thread` exists for the rare cases where you need a dedicated thread with specific properties (high stack size, COM apartment state, foreground vs background semantics).

---

## Why the split

Threads were the only option in early .NET. Creating one took ~1 ms and used ~1 MB of memory. Running 1000 tasks meant 1000 threads = 1 GB of stack + lots of kernel scheduling overhead.

The **thread pool** was added to amortize that: keep a small pool of long-lived threads, hand them work items as they arrive. Each work item finishes quickly (or yields to the next), so a few threads service many items.

`Task` is the API over the thread pool. `async/await` is the language sugar over `Task`. Together, they let you write "1000 concurrent HTTP requests" without creating 1000 threads.

---

## When to use each

### Use `Task` (or `async/await`) when:
- The work is short or async (network, file, database).
- You want fan-out parallelism.
- You want a cancellation/continuation/exception model.
- Default for everything.

### Use `Thread` directly when:
- You need a dedicated long-running thread (e.g., a game loop, a UI thread surrogate).
- You need a non-default stack size (`new Thread(_, maxStackSize: 8 * 1024 * 1024)`).
- You need COM apartment state (STA for old WinForms/WPF/Office automation).
- You need a **foreground** thread (one that keeps the process alive — opposite of `IsBackground = true`).

### Use `ThreadPool.QueueUserWorkItem` when:
- You want raw thread pool work without `Task` semantics.
- Almost never — `Task.Run` is the modern equivalent and gives you cancellation, exception propagation, etc.

---

## Creating a Thread

```csharp
var t = new Thread(() => {
    Console.WriteLine("Running on a new thread");
});
t.Name = "MyWorker";
t.IsBackground = true;     // dies with the process
t.Start();
t.Join();                   // wait for it
```

`IsBackground = true` means the thread won't keep the process alive. `false` (default) keeps the process running until the thread exits. Important for non-daemon threads.

For threads taking arguments:
```csharp
var t = new Thread(arg => { var n = (int)arg!; ... });
t.Start(42);   // single parameter, boxed
```

Or with a delegate:
```csharp
var t = new Thread(() => { Process(myData); });   // capture via closure
t.Start();
```

---

## Creating a Task

```csharp
var task = Task.Run(() => DoWork());        // CPU-bound work on thread pool
var task2 = DoWorkAsync();                   // async method (no Task.Run!)
await Task.Run(() => HeavyCompute());        // wait without blocking
```

`Task.Run` queues the lambda to the thread pool. The returned Task lets you `await`, attach continuations, or handle exceptions.

For I/O-bound work, **don't wrap it in Task.Run** — async methods already release the thread during I/O. `Task.Run(() => httpClient.GetAsync(...))` is wasteful — it grabs a thread pool thread just to immediately await another thread pool thread.

---

## The thread pool

The CLR maintains a pool of worker threads, started at:
- Min count = Environment.ProcessorCount (logical CPUs).
- Max count = high (default 32k+).

When you `Task.Run(workItem)`:
1. The work item is added to a queue.
2. A pool thread picks it up and runs it.
3. When it finishes, the thread goes back to the pool.

For I/O completions (async file/network), there's a separate **I/O completion port** mechanism. When an async I/O finishes, the runtime grabs a thread pool thread to run the continuation.

The pool **grows** lazily when its work queue stays full — but slowly (one new thread per ~500 ms, by default) to avoid thrashing. This is called **thread-pool hill climbing**.

If your code blocks pool threads (sync I/O, `.Wait()`, lock contention), the pool can starve and grow slowly — leading to mysterious throughput collapses. **Don't block pool threads.**

---

## Cost comparison

For 10,000 concurrent operations:

| Approach | Thread count | Memory | Time to start |
|---|---|---|---|
| `new Thread(...).Start()` × 10,000 | 10,000 | ~10 GB | seconds |
| `Task.Run(...)` × 10,000 | ~ProcessorCount | ~MBs | milliseconds |
| `await Task.WhenAll(10,000 async I/Os)` | ~ProcessorCount | minimal | milliseconds |

`await Task.WhenAll` of async operations doesn't need a thread per operation — while each is waiting for I/O, the threads are free for other work. **A handful of threads can service thousands of async operations.**

This is the modern .NET concurrency model. Async-everywhere is the default; explicit threads are rare.

---

## When to use Task.Run

`Task.Run` exists for **CPU-bound** work you want offloaded. Not for I/O.

```csharp
// ✓ CPU-bound — moves work off the calling thread
var task = Task.Run(() => ProcessBigImage(image));

// ✗ I/O-bound — pointless, the GetAsync already releases the thread
var task = Task.Run(() => httpClient.GetAsync("https://example.com"));
```

For I/O, just call the async method directly:
```csharp
var task = httpClient.GetAsync("https://example.com");
```

The asymmetry: `Task.Run` is for "I have a sync, expensive call I want to make non-blocking." For methods already async, no Task.Run.

---

## Background threads vs foreground threads

Each .NET process has a main thread (foreground by default). When all foreground threads finish, the process exits — even if background threads are still running.

```csharp
var t = new Thread(() => { Thread.Sleep(10_000); Console.WriteLine("done"); });
t.IsBackground = false;   // process waits for this thread to finish
t.Start();
// main thread exits — but the foreground thread keeps the process alive
```

```csharp
t.IsBackground = true;
t.Start();
// main thread exits — background thread is killed mid-Sleep
```

**Thread pool threads are always background.** A `Task.Run` operation never keeps the process alive. If you want long-running work that must finish, use a `Thread` with `IsBackground = false`, or set up a graceful shutdown.

---

## ThreadStatic vs AsyncLocal

For per-thread data:

```csharp
[ThreadStatic] private static int _threadLocalCounter;   // per thread

// Issue: doesn't flow across await
async Task M() {
    _threadLocalCounter = 5;
    await Task.Delay(100);
    Console.WriteLine(_threadLocalCounter);   // ⚠ — different thread, value is 0!
}
```

`[ThreadStatic]` is tied to the OS thread. After `await`, the continuation may run on a different thread → the value is lost.

**Use `AsyncLocal<T>`** for data that should follow the logical flow of async operations:

```csharp
private static readonly AsyncLocal<int> _counter = new();

async Task M() {
    _counter.Value = 5;
    await Task.Delay(100);
    Console.WriteLine(_counter.Value);   // ✓ — 5, regardless of thread
}
```

`AsyncLocal<T>` flows through the `ExecutionContext` — captured at each await and restored on the continuation. Used by `ILogger`'s scopes, `Activity` (distributed tracing), `HttpContext.Current`, etc.

---

## SynchronizationContext (preview)

When a thread is associated with a synchronization context (UI thread on WPF/WinForms; old ASP.NET request thread), `await` by default **captures the context** and resumes the continuation on it.

```csharp
// On the UI thread:
public async void ButtonClick() {
    var data = await httpClient.GetStringAsync(url);
    label.Text = data;     // ← runs back on UI thread automatically
}
```

This is the magic that makes UI code work. We'll dive deep in [§06 ConfigureAwait](06-ConfigureAwait.md).

ASP.NET Core, console apps, and worker services have NO SynchronizationContext — continuations resume on any free thread pool thread.

---

## Internals — what a Thread actually is

A .NET `Thread` wraps an OS thread:
- Linux: pthread.
- Windows: WIN32 thread.

Each has:
- A stack (default 1 MB on x64, configurable via `Thread` constructor).
- Thread-local storage slots.
- A reference to the CLR's per-thread state (allocation context, exception unwind state, etc.).

Creating one calls `CreateThread` / `pthread_create`. Each new OS thread:
- Allocates ~1 MB of virtual memory for the stack.
- Adds to the kernel's scheduler list.
- Costs ~100-500 μs on most systems.

For thousands of threads, this overhead becomes the bottleneck. Hence the thread pool.

---

## Internals — what the thread pool actually is

The CLR thread pool has:
- A **global queue** of work items.
- A **local queue** per worker thread (work-stealing).
- A separate **IO completion port** queue.
- A hill-climbing algorithm that grows the pool based on throughput.

When you call `Task.Run`:
1. Creates a Task object.
2. Wraps your delegate in a thread-pool work item.
3. Queues it (to the calling thread's local queue if it's already a pool thread, else the global queue).
4. A worker thread picks it up.

Workers prefer their own local queue, then steal from others, then fall back to the global queue.

When a worker finishes a work item:
- If queue empty, sleeps briefly, then exits if no work arrives.
- Hill-climbing decides whether to add more threads.

For deeply async code, the same few worker threads service many tasks — each `await` releases the thread for other work.

---

## Common bugs

### Blocking pool threads with sync I/O

```csharp
public async Task<string> Bad() {
    return await Task.Run(() => {
        Thread.Sleep(10_000);          // ⚠ blocks a pool thread for 10s
        return File.ReadAllText("...");  // ⚠ also blocks
    });
}
```

Pool can starve. Use async APIs (`Task.Delay`, `File.ReadAllTextAsync`).

### Forgetting `IsBackground`

```csharp
new Thread(LongLoop).Start();   // foreground — keeps process alive
```

In a console app, this can prevent the process from exiting. Set `IsBackground = true` if you want the thread to die with the process.

### Sync-over-async

```csharp
public string Get() => GetAsync().Result;   // ⚠
```

`.Result` blocks the calling thread until the task completes. If the await captures a SynchronizationContext that's now blocked, deadlock. Even in non-deadlock cases, it wastes a thread.

[§17 CommonAsyncBugs](17-CommonAsyncBugs.md) covers all the variants.

---

## When you really need a Thread

- A game's main loop — pinned to one thread, never yields.
- A dedicated I/O thread for a specific device with its own protocol.
- COM interop requiring STA apartment.
- Long-running compute that should NOT share the thread pool.

For everything else, async/await + Task is the answer.

---

## Performance summary

| | Thread | Task (via pool) | Async I/O |
|---|---|---|---|
| Cost to start | ~100-500 μs | ~1 μs | ~1 μs |
| Memory per | ~1 MB | ~200 bytes | ~200 bytes |
| Concurrent count practical | hundreds | thousands | tens of thousands |
| Sync APIs | OK to use | block pool — bad | n/a |
| Async APIs | wasteful | fine | optimal |

→ Next: [02-AsyncAwaitFundamentals.md](02-AsyncAwaitFundamentals.md)
