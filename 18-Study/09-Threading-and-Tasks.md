# 09 — Threading & Tasks ⭐ (high-frequency)

## ⚡ 30-second answer

A **thread** is an OS-scheduled unit of execution (expensive: ~1MB stack, kernel scheduling). A **`Task`** is a higher-level promise of future work, usually run on the **thread pool** — you almost never create raw `Thread`s anymore; you use `Task`/`async`. The thread pool **reuses** a managed set of worker threads (with work-stealing queues and a hill-climbing algorithm that adds/removes threads). Use `Task.Run` for **CPU-bound** offloading; use **`async`/`await`** for **I/O-bound** work (which frees the thread entirely — no thread is blocked waiting).

---

## Core mechanics

**Thread vs ThreadPool vs Task**:
```csharp
new Thread(Work).Start();              // dedicated OS thread — rare; for long-running/blocking work
ThreadPool.QueueUserWorkItem(_ => …);  // low-level pool work
var t = Task.Run(() => Compute());     // CPU-bound work on the pool — the common case
await SomeIoAsync();                    // I/O-bound — NO thread blocked during the wait
```

**Thread pool**: a managed pool of reusable worker threads.
- **Work-stealing**: each thread has a local queue; idle threads steal from others' queues → load balancing.
- **Hill climbing**: injects/retires threads based on throughput. Injection is **slow/throttled** — bursts of blocking work cause **starvation** (work queued, few threads, latency spikes) ([12](12-Concurrent-Parallel-AsyncBugs.md)).
- Separate handling for **I/O completions** (IOCP on Windows).

**Task lifecycle / composition**:
```csharp
Task<int> t = Task.Run(() => 42);
await t;                               // observe result/exception
await Task.WhenAll(t1, t2, t3);        // wait for all (runs concurrently)
var first = await Task.WhenAny(t1, t2);// first to finish
await foreach (var done in Task.WhenEach(tasks)) { … }  // .NET 9 — process as they complete
```

**`TaskCompletionSource<T>`** — create a `Task` you complete manually (bridge a callback/event into the await world):
```csharp
var tcs = new TaskCompletionSource<int>();
button.Click += (_,__) => tcs.TrySetResult(42);
int result = await tcs.Task;           // awaits until SetResult/SetException/SetCanceled
```

---

## Comparison tables

| | `Thread` | `Task` (on pool) |
|---|---|---|
| Cost | heavy (OS thread, ~1MB stack) | light (queued work, reused thread) |
| Returns a result | no (manual) | `Task<T>` |
| Exceptions | crash unless handled | captured on the Task |
| Use | long-running/blocking, must-not-be-pool | almost everything else |

| Work type | Use | Why |
|---|---|---|
| **CPU-bound** (compute, parse) | `Task.Run` | offload to a pool thread, keep UI/request thread free |
| **I/O-bound** (DB, HTTP, file) | `async`/`await` (no `Task.Run`) | no thread blocked while waiting — scalability |
| long-running, blocking | dedicated `Thread` or `TaskCreationOptions.LongRunning` | don't starve the pool |

---

## 🪤 Traps & gotchas

- **`Task.Run` for I/O**: wrapping `await httpClient.GetAsync()` in `Task.Run` just burns a pool thread to start async work — pointless. Async I/O needs no thread while waiting. Only offload **CPU** work.
- **Blocking the pool** (`.Result`/`.Wait()`/synchronous I/O on pool threads) → **thread-pool starvation**: requests queue while the pool slowly injects threads → latency cliff under load ([10](10-AsyncAwait.md), [12](12-Concurrent-Parallel-AsyncBugs.md)).
- **Unobserved task exceptions**: a faulted `Task` you never await loses its exception (fire-and-forget). Always await or handle.
- **`Task.WhenAll` exceptions**: if multiple tasks throw, `await` surfaces only the **first**; check `task.Exception?.InnerExceptions` for all.
- **`async` lambda passed to a `void`-returning API** becomes `async void` (uncatchable) — `Parallel.ForEach(async …)` is a bug.
- **Creating threads in a loop** → resource exhaustion. Use the pool/`Task`.
- **`Task` is hot by default** — `Task.Run`/most factory methods start immediately; there's no "start later" unless you use `new Task(...)` (rare) or cold async methods.

---

## ❓ Likely questions

**Q: Thread vs Task?**
A: A thread is a heavy OS execution unit; a Task is a lightweight promise of work, usually scheduled on the reused thread pool. Prefer Tasks/async; create raw threads only for long-running blocking work.

**Q: When `Task.Run` vs `async/await`?**
A: `Task.Run` offloads **CPU-bound** work to a pool thread. `async/await` is for **I/O-bound** work and doesn't consume a thread while waiting. Don't `Task.Run` async I/O.

**Q: What is thread-pool starvation?**
A: When pool threads are blocked (e.g., `.Result` on async work) faster than the pool injects new ones, queued work stalls and latency spikes. Fix by going async all the way.

**Q: How does the thread pool work?**
A: A managed set of reusable worker threads with per-thread local queues and work-stealing for balance; a hill-climbing heuristic adjusts thread count by throughput; I/O completions use a separate mechanism.

**Q: `WhenAll` vs `WhenAny` vs `WaitAll`?**
A: `Task.WhenAll`/`WhenAny` are **awaitable** (non-blocking) — all / first-to-complete. `Task.WaitAll`/`WaitAny` are **blocking** (avoid in async code).

**Q: What's `TaskCompletionSource`?**
A: A handle to manually create and complete a Task — used to wrap callbacks/events/external signals into an awaitable.

**Q: How do you run several async operations concurrently?**
A: Start them (don't await individually), then `await Task.WhenAll(...)`. Starting then awaiting one-by-one serializes them.

---

## 🎓 Senior Extra

- **`Task` vs `ValueTask`**: `ValueTask<T>` avoids a `Task` allocation when results often complete synchronously (cache hits) — but **don't await it twice** or access `.Result` before completion ([10](10-AsyncAwait.md)).
- **`TaskScheduler`**: Tasks run on a scheduler (default = thread pool). UI frameworks have a scheduler bound to the UI thread (`FromCurrentSynchronizationContext`) — relevant to continuations and `ConfigureAwait` ([10](10-AsyncAwait.md)).
- **`TaskCreationOptions.LongRunning`** hints a dedicated thread (off the pool) so a long blocking loop doesn't occupy a pool thread.
- **`Channel<T>`** is the modern producer/consumer primitive — prefer it over `BlockingCollection` for async pipelines ([12](12-Concurrent-Parallel-AsyncBugs.md)).
- **`Parallel.ForEachAsync`** (.NET 6) runs async work over a collection with a concurrency limit — the right tool for "call this API for 10k items, 20 at a time" (vs `Task.WhenAll` over 10k tasks, which floods).
- **Cancellation & timeouts** compose via `CancellationTokenSource` (linked tokens, `CancelAfter`) — always flow a token into tasks ([10](10-AsyncAwait.md)).
- **`ThreadPool.SetMinThreads`** can paper over starvation temporarily but the real fix is removing blocking; measure pool queue length with `dotnet-counters` ([21](21-Deployment-Perf-Tooling.md)).
- **Thread affinity**: pool threads aren't stable — never store thread-local state expecting the same thread across awaits, and never touch UI controls off the UI thread.

→ Deeper: [`../CSharpBook/08-Concurrency/`](../CSharpBook/08-Concurrency/README.md), [`../DotNetBook/01-Runtime/08-Threading.md`](../DotNetBook/01-Runtime/README.md)
